# RescueOS — Architecture (C4)

This document captures the **C4 architecture diagrams** for **RescueOS**, aligned with the agreed technical approach:

* Multi-tenant (organisations, teams, sub-teams)
* Azure Container Apps (ACA)
* Keycloak for authentication and federation
* Dedicated Authorization (AuthZ) service with team-defined roles and granular permissions
* API-first, microservice-based design

---

## C4 Level 1 — System Context

```mermaid
flowchart LR
  User[Volunteer / Crew / Controller] --> Web[RescueOS Web App]
  User --> Mobile[RescueOS Mobile App]

  Web --> API[RescueOS Platform API]
  Mobile --> API

  ExtIdP[External Identity Providers] --> KC[Keycloak]
  KC --> API

  API --> Notify[SMS/Email/Push Providers]
  API --> Maps[Mapping / GIS Services]
  API --> Weather[Weather / River / Met data feeds]
```

**Intent**
RescueOS is accessed via web and mobile applications. Users authenticate via **Keycloak**, which may federate to external identity providers. RescueOS integrates with external notification services, mapping/GIS providers, and environmental data sources.

---

## C4 Level 2 — Container Diagram

```mermaid
flowchart TB
  subgraph Client[Clients]
    Web[Web App]
    Mobile[Mobile App]
  end

  subgraph Edge[Azure Edge]
    APIM[API Gateway]
    WAF[Front Door / WAF]
  end

  subgraph Identity[Identity]
    KC[Keycloak OIDC / OAuth Federation]
  end

  subgraph Platform[RescueOS on Azure Container Apps]
    BFF[BFF / API Aggregator]
    Inc[Incident Service]
    Task[Tasking & Dispatch Service]
    Team[Teams & Roster Service]
    Person[People / Profiles Service]
    Train[Training & Competency Service]
    Asset[Assets & Equipment Service]
    Log[Operations Log / Audit Service]
    AuthZ["Authorization Service (PDP + Policy Store)"]
    NotifySvc[Notification Service]
    FileSvc[Files / Media Service]
    Search[Search / Index Service]
  end

  subgraph Messaging[Async / Integration]
    Bus[Azure Service Bus]
    Eg["Event Grid (optional)"]
  end

  subgraph Data[Data Stores]
    SQL[(Azure SQL / PostgreSQL)]
    Cosmos[Cosmos DB]
    Redis[(Redis Cache)]
    Blob[(Blob Storage)]
  end

  subgraph Observability[Observability]
    OTel[OpenTelemetry Collector]
    Appi[Application Insights]
    Logs[Log Analytics]
  end

  Web --> WAF --> APIM
  Mobile --> WAF --> APIM
  APIM --> KC
  APIM --> BFF

  BFF --> Inc
  BFF --> Task
  BFF --> Team
  BFF --> Person
  BFF --> Train
  BFF --> Asset
  BFF --> Search
  BFF --> FileSvc
  BFF --> NotifySvc

  Inc --> AuthZ
  Task --> AuthZ
  Team --> AuthZ
  Person --> AuthZ
  Train --> AuthZ
  Asset --> AuthZ
  Log --> AuthZ

  Inc <--> SQL
  Team <--> SQL
  Person <--> SQL
  Train <--> SQL
  Asset <--> SQL
  AuthZ <--> SQL
  Search <--> Cosmos
  FileSvc <--> Blob
  BFF <--> Redis

  Inc --> Bus
  Task --> Bus
  Team --> Bus
  Person --> Bus
  Train --> Bus
  Asset --> Bus

  Bus --> NotifySvc
  Bus --> Search

  BFF --> OTel
  Inc --> OTel
  Task --> OTel
  Team --> OTel
  Person --> OTel
  Train --> OTel
  Asset --> OTel
  AuthZ --> OTel
  OTel --> Appi
  Appi --> Logs
```

**Notes**

* Tenant isolation is logical, enforced via `tenant_id` and authorization policy rather than infrastructure isolation.
* Keycloak is responsible only for authentication and federation.
* All authorization decisions are centralised via the AuthZ service.
* Event-driven patterns decouple operational workflows (e.g. incident → notification → reporting).

---

## C4 Level 3 — Authorization Service (AuthZ) Components

```mermaid
flowchart TB
  subgraph AuthZ[Authorization Service]
    API["AuthZ Management API (Roles, Permissions, Assignments)"]
    PDP[Policy Decision Point]
    Store["(Policy Store Roles, Permissions, Memberships)"]
    Cache["(Decision Cache Short TTL)"]
    Audit[Decision Audit Emitter]
  end

  subgraph Callers[Calling Services]
    Inc[Incident Service]
    Team[Teams Service]
    Task[Dispatch Service]
    BFF[BFF / Gateway Layer]
  end

  subgraph Identity[Identity]
    KC[Keycloak
JWT Access Tokens]
  end

  subgraph Messaging[Events]
    Bus[Service Bus]
  end

  BFF -->|JWT| Inc
  BFF -->|JWT| Team
  BFF -->|JWT| Task
  Inc -->|"Authorize (action, resource, tenant)"| PDP
  Team -->|"Authorize(...)"| PDP
  Task -->|"Authorize(...)"| PDP

  PDP --> Cache
  PDP --> Store

  PDP -->|allow / deny| Inc
  PDP -->|allow / deny| Team
  PDP -->|allow / deny| Task

  API --> Store
  API -->|policy change events| Bus
  Audit --> Bus

  KC --> BFF
```

### AuthZ Responsibilities

* Maintain tenant-defined roles and granular permissions
* Map users to roles within tenant/team scopes
* Evaluate authorization decisions consistently across services
* Provide auditable allow/deny decisions

### Policy Enforcement Points (PEP)

* Implemented as middleware in each service (e.g. Laravel middleware)
* Extract user identity and tenant context from request
* Call AuthZ PDP or use cached decisions
* Enforce allow/deny and apply scope restrictions

---

## Summary

These C4 diagrams describe how **RescueOS**:

* Supports multi-tenant SAR organisations
* Separates authentication from authorization
* Allows teams to define their own roles and permissions
* Scales cleanly on Azure Container Apps
* Remains observable, auditable, and maintainable

This document should be treated as the **baseline architectural reference** for RescueOS.
