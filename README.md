# StudentPulse
StudentPulse is an application where an instructor creates students and updates their status or grade, while the system broadcasts events.

## 🏗 Project Structure (Vertical Slice Architecture)
The project is organized following the principles of Vertical Slice Architecture (VSA). Instead of the traditional separation into technical layers (controllers, services, repositories), the code is grouped by specific business scenarios.
```text
📁 StudentPulse.sln
│
├── 📁 StudentPulse.AppHost          # Aspire project to run the entire infrastructure
│    └── Program.cs                   # Configures the startup for API, React, MongoDB, Kafka, and Azure Function
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
│    ├── StudentCreatedFunction.cs    # Listens to Kafka and sends notifications to Google Chat and Email
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
          │    │    └── CreateStudent.cs    # (Command + Handler). Validates, saves to DB, sends StudentCreated event
          │    │    └── StudentCreated.cs   # 📦 Event model for Kafka
          │    │
          │    ├── 📁 GetList
          │    │    └── GetStudents.cs      # (Query + Handler). Simply reads the list of students from DB (Active / Graduated / Dropped)
          │    ├── 📁 GetById
          │    │    └── GetStudent.cs       # (Query + Handler). Reads a single student from DB by ID
          │    │
          │    ├── 📁 Delete
          │    │    └── DeleteStudent.cs    # (Command + Handler). Checks Admin role, deletes from DB, sends StudentDeleted to Kafka
          │    │    └── StudentDeleted.cs       # 📦 Event model for Kafka
          │    │
          │    └── 📁 Update
          │         └── UpdateStudent.cs    # Updates the student's Status or Score
          │         └── StudentStatusChanged.cs   # 📦 Event model for Kafka (record)
          │         └── StudentScoreChanged.cs    # 📦 Event model for Kafka (record)
          │         └── StudentProfileUpdated.cs  # 📦 Event model for  Kafka (record)
          │
          └── 📁 Settings
               └── 📁 ChangeStorage
                    ├── ChangeStorage.cs    # (Command + Handler). Accepts an HTTP request and switches the active database provider.
                    └── StorageChanged.cs   # 📦 (Optionaly) Audit event for Kafka, if a change in settings needs to be recorded.
