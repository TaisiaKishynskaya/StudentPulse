# StudentPulse
StudentPulse is an application where an instructor creates students and updates their status or grade, while the system broadcasts events.

## 🔄 High-Level Architecture & Data Flow
All infrastructure components, databases, and message brokers are orchestrated locally via **.NET Aspire** for a seamless one-click startup.
```text
                   ┌─────────────────┐
                   │      React      │
                   └────────┬────────┘
                            │ HTTP
                            ▼
                   ┌─────────────────┐
                   │ ASP.NET Web API │
                   │     Identity    │
                   └───────┬─────────┘
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
      MongoDB / Cosmos DB         Kafka / Event Bus
             │                           │
             │                        Events
             │                           │
             │                           ▼
             │                     Azure Function
             │                           │
             │                      Google Chat
             │                         Email
             │
             └───────────────────────────

             Everything runs through Aspire
```

## 🎨 This project brings together a set of modern architectural patterns.
* **Vertical Slice Architecture (VSA)** — all backend code is grouped by business features rather than technical layers.
* **Event-Driven Architecture (EDA)** — when the API state changes, it does not synchronously call third-party services but instead publishes domain events to a Kafka message bus.
* **Lightweight CQRS / CQS (Command Query Separation)** - within vertical slices, the logic is clearly separated into Commands (write/modify) and Queries (read).
* **Mediator Pattern (через Wolverine)** — an HTTP request arrives at a Minimal API (or controller), which simply takes the request object and tells Wolverine: "Find someone who knows how to handle this." Wolverine locates the appropriate handler in the folder and passes the request to it.
* **Repository Pattern + Strategy Pattern** — the Infrastructure\Storage hides DB details behind a common IStudentRepository interface to keep the slice code clean. The Settings page uses the Strategy pattern to dynamically switch between MongoStudentRepository and CosmosStudentRepository. As a result, the feature slice simply calls _repository.DeleteAsync(id) without caring which database is currently active — the infrastructure layer intercepts the call and seamlessly routes it to either MongoDB or Cosmos DB.
* **Serverless Computing / Background Offloading** — the StudentPulse.Notifications project, based on Azure Functions, operates as an isolated event handler.


## 🏗 Project Structure (Vertical Slice Architecture)
The project is organized following the principles of Vertical Slice Architecture (VSA). Instead of the traditional separation into technical layers (controllers, services, repositories), the code is grouped by specific business scenarios.
```text
📁 StudentPulse.sln
│
├── 📁 StudentPulse.AppHost          # Aspire project to run the entire infrastructure
│    └── Program.cs                  # Configures the startup for API, React, MongoDB, Kafka, and Azure Function
│
├── 📁 StudentPulse.ServiceDefaults  # Standard Aspire project (Telemetry, HealthChecks)
│
├── 📁 StudentPulse.Frontend         # React + TypeScript + Vite + Material UI
│    ├── 📁 src
│    │    ├── 📁 components          # Table, Dialog, Button, Select (MUI)
│    │    └── 📁 pages               # Login, Students, Settings
│    └── package.json
│
├── 📁 StudentPulse.Notifications    # Azure Functions project
│    ├── StudentCreatedFunction.cs   # Listens to Kafka and sends notifications to Google Chat and Email
│    └── host.json
│
└── 📁 StudentPulse.Api              # ASP.NET Web API project
     │
     ├── 📁 Infrastructure           # Technical configurations
     │    ├── 📁 Auth                # Identity settings (Microsoft Entra / Auth0) and Admin / Teacher roles
     │    ├── 📁 Storage             # Implementations for MongoStudentRepository and CosmosStudentRepository
     │    └── 📁 Messaging           # Wolverine connection settings for Kafka
     │
     └── 📁 Features                 # Vertical Slices (VSA)
          │
          ├── 📁 Students            # Slices for the Student entity
          │    │
          │    ├── 📁 Create
          │    │    ├── CreateStudent.cs         # (Command + Handler). Validates, saves to DB, sends StudentCreated event
          │    │    └── StudentCreated.cs        # 📦 Event model for Kafka
          │    │
          │    ├── 📁 GetList
          │    │    └── GetStudents.cs           # (Query + Handler). Simply reads the list of students from DB (Active / Graduated / Dropped)
          │    ├── 📁 GetById
          │    │    └── GetStudent.cs            # (Query + Handler). Reads a single student from DB by ID
          │    │
          │    ├── 📁 Delete
          │    │    ├── DeleteStudent.cs         # (Command + Handler). Checks Admin role, deletes from DB, sends StudentDeleted to Kafka
          │    │    └── StudentDeleted.cs        # 📦 Event model for Kafka
          │    │
          │    └── 📁 Update
          │         ├── UpdateStudent.cs         # Updates the student's Status or Score
          │         ├── StudentStatusChanged.cs  # 📦 Event model for Kafka (record)
          │         ├── StudentScoreChanged.cs   # 📦 Event model for Kafka (record)
          │         └── StudentProfileUpdated.cs # 📦 Event model for Kafka (record)
          │
          └── 📁 Settings
               └── 📁 ChangeStorage
                    ├── ChangeStorage.cs         # (Command + Handler). Accepts an HTTP request and switches the active database provider.
                    └── StorageChanged.cs        # 📦 (Optional) Audit event for Kafka, if a change in settings needs to be recorded.

```
### 🧩 Core Components
* **StudentPulse.AppHost (The Conductor):** This is the single entry point for launching the entire system. It reads `Program.cs`, understands the dependencies, and spins up the entire local infrastructure with a single click.
* **StudentPulse.ServiceDefaults (The Life Support System):** An auxiliary .NET Aspire project that acts as a "medic" and "spy" for the services. It automatically injects base configurations (Health Checks and OpenTelemetry) into the API. Thanks to this project, the Aspire dashboard can draw beautiful call graphs and display service statuses.
* **StudentPulse.Frontend (The React Client):** This is where the UI is rendered. The frontend doesn't compute anything itself or interact with the database—it only draws the interface and sends HTTP requests to the API.
* **StudentPulse.Notifications (The Background Worker / Courier):** An Azure Functions-based microservice dedicated specifically to slow background tasks. To avoid making the user wait while an email or Google Chat message is sent, the main API simply drops an event into Kafka instantly. This project "listens" to Kafka, intercepts the event, and performs the long-running task of sending notifications completely invisibly to the frontend.
* **StudentPulse.Api (The Core Backend):** Receives requests from the React application. It is strictly divided into two areas of responsibility:
* **Infrastructure:** The hidden "plumbing" of the application. This is where database settings, message broker configurations, and access control logic (Identity) reside.
* **Features:** The code is split not by technical layers/tiers, but by specific business actions. Each folder is an isolated "pipe" that takes one specific HTTP request from the user, validates it, saves changes to the DB via the Infrastructure, and spits out a ready-made domain event into Kafka.
