# Jenkins CI/CD Platform — Architecture

[← Back to README](../README.md) | [Implementation](implementation.md) | [Validation](validation.md) | [Limitations & Future Work](limitations-and-future-work.md)

---

## 1. Architecture Overview

This project demonstrates a Jenkins-based CI/CD architecture for an existing Java application workload.

The architecture connects source control, continuous integration, quality validation, artifact management, containerization, container registry storage, and container deployment into one delivery flow.

The core architecture is:

```text
                         ┌──────────────┐
                         │   Developer  │
                         └──────┬───────┘
                                │
                                │ Code Push
                                ▼
                         ┌──────────────┐
                         │    GitHub    │
                         └──────┬───────┘
                                │
                         Webhook / Trigger
                                │
                                ▼
                    ┌──────────────────────┐
                    │       Jenkins        │
                    │                      │
                    │ Fetch Code           │
                    │ Build                │
                    │ Unit Test            │
                    │ Checkstyle           │
                    │ SonarQube Analysis   │
                    │ Quality Gate          │
                    │ Publish Artifact      │
                    │ Docker Build         │
                    │ Push Image            │
                    │ Deploy                │
                    └──────┬─────────┬─────┘
                           │         │
                           │         │
                           ▼         ▼
                     ┌─────────┐ ┌─────────┐
                     │  Nexus  │ │   ECR   │
                     │ Artifact│ │Container│
                     │ Storage │ │ Registry │
                     └─────────┘ └────┬────┘
                                      │
                                      ▼
                                ┌───────────┐
                                │    ECS    │
                                │  Service  │
                                └─────┬─────┘
                                      │
                                      ▼
                              Running Container
```

The architecture separates the responsibilities of the major systems rather than treating Jenkins as the implementation of every DevOps function.

---

## 2. Application Ownership Boundary

The VProfile application is the workload used by this project.

The architecture therefore represents:

```text
Existing Application
        +
DevOps / CI/CD Platform
```

and not:

```text
Application Development
        +
DevOps / CI/CD Platform
```

The project focuses on the engineering performed around the application:

- Source integration
- Build automation
- Testing
- Code-quality analysis
- Artifact management
- Containerization
- Image publishing
- Container deployment
- Pipeline automation
- Triggering
- Notifications
- Jenkins security

The application business logic and Java application development are outside the demonstrated ownership boundary.

---

# 3. Architectural Responsibility Model

Each major component has a distinct responsibility.

| Component | Primary Responsibility |
|---|---|
| Developer | Creates and pushes source changes |
| GitHub | Source-code repository and source-change event |
| Jenkins | CI/CD orchestration |
| Maven | Build and test execution |
| Checkstyle | Source-code style analysis |
| SonarQube | Static code analysis |
| Quality Gate | Controls progression based on analysis result |
| Nexus | Versioned application artifact storage |
| Docker | Packages the application into a container image |
| Amazon ECR | Stores container images |
| Amazon ECS | Runs and manages the containerized workload |
| Slack | Pipeline-result notification |

This separation is important because the pipeline is an orchestration layer connecting specialized systems.

---

# 4. End-to-End Delivery Flow

The architecture can be understood as a sequence of controlled transitions:

```text
Source
  ↓
Build
  ↓
Test
  ↓
Analyze
  ↓
Quality Gate
  ↓
Artifact
  ↓
Container Image
  ↓
Registry
  ↓
Deployment
```

Each stage produces information or an artifact consumed by the next stage.

---

## 4.1 Source Control

GitHub acts as the source-control system.

The Jenkins pipeline retrieves the application source from the configured repository and branch.

The demonstrated Docker CI/CD pipeline uses the `docker` branch as its source branch.

```text
GitHub Repository
       │
       │
       ▼
docker branch
       │
       ▼
Jenkins
```

---

## 4.2 Jenkins Orchestration

Jenkins is the central orchestration layer.

It coordinates the pipeline stages but delegates specialized work to external tools.

Conceptually:

```text
                     Jenkins
                        │
       ┌────────────────┼────────────────┐
       │                │                │
       ▼                ▼                ▼
     Maven          SonarQube          Nexus
       │                │                │
       ▼                ▼                ▼
    Build/Test       Analysis        Artifact
       │
       ▼
     Docker
       │
       ▼
      ECR
       │
       ▼
      ECS
```

Jenkins therefore acts as the workflow controller rather than the storage location for every artifact.

---

# 5. Continuous Integration Architecture

The CI portion of the project is:

```text
GitHub
   ↓
Fetch Code
   ↓
Maven Build
   ↓
Unit Test
   ↓
Checkstyle Analysis
   ↓
SonarQube Analysis
   ↓
Quality Gate
   ↓
Publish to Nexus
   ↓
Build Docker Image
   ↓
Push Docker Image to ECR
```

The CI pipeline is designed so that later stages depend on the successful completion of earlier stages.

---

## 5.1 Build Boundary

Maven builds the Java application and produces the application artifact.

Conceptually:

```text
Source Code
     ↓
   Maven
     ↓
Application Artifact
```

The artifact becomes an input to later stages.

---

## 5.2 Testing Boundary

Unit tests execute as part of the Maven workflow.

```text
Build
  ↓
Unit Tests
  ↓
Pass / Fail
```

A failed build or test prevents the pipeline from progressing normally into subsequent stages.

---

## 5.3 Code Quality Boundary

The architecture contains two related quality mechanisms:

```text
Source Code
    ↓
Checkstyle
    ↓
SonarQube Analysis
    ↓
Quality Gate
```

Checkstyle produces code-style analysis information.

SonarQube performs broader static analysis and receives supporting information such as test reports, coverage information, and Checkstyle results.

---

## 5.4 Quality Gate

The Quality Gate represents an explicit decision boundary.

```text
SonarQube Analysis
        ↓
   Quality Gate
      /     \
    PASS     FAIL
     │         │
     ▼         ▼
 Continue    Stop
```

This prevents the pipeline from blindly continuing after an unsuccessful quality evaluation.

The Jenkins pipeline waits for the SonarQube Quality Gate result and is configured to abort when the gate fails.

---

# 6. Artifact Management Architecture

The project separates the application artifact from the Jenkins workspace.

```text
Jenkins Workspace
       │
       │ Build Artifact
       ▼
     Nexus
       │
       │ Versioned Artifact
       ▼
 Central Artifact Storage
```

Nexus acts as the centralized artifact repository.

The demonstrated pipeline publishes the generated WAR artifact to the `vprofile-repo` repository with build-specific version information.

This creates a distinction between:

```text
Build Workspace
      ≠
Artifact Repository
```

The Jenkins workspace is temporary execution space, while Nexus provides centralized artifact storage.

---

# 7. Containerization Architecture

The Docker stage changes the deployable unit.

Before containerization:

```text
Source
  ↓
Maven
  ↓
WAR Artifact
```

After containerization:

```text
WAR Artifact
     ↓
Docker Build
     ↓
Docker Image
```

The Docker image becomes the deployable unit for the ECS portion of the architecture.

---

## 7.1 Docker Image Flow

```text
Application Source
       ↓
     Maven
       ↓
   WAR Artifact
       ↓
 Dockerfile
       ↓
 Docker Build
       ↓
 Docker Image
       ↓
 Image Tag
```

The demonstrated pipeline uses Jenkins build information to create build-specific image tags and also maintains a `latest` tag.

Conceptually:

```text
vprofileappimg:<build-number>
vprofileappimg:latest
```

The build-specific tag provides a relationship between a Jenkins build and a container image.

---

# 8. Amazon ECR Architecture

Amazon Elastic Container Registry acts as the container image registry.

```text
Jenkins
   │
   │ Docker Push
   ▼
Amazon ECR
   │
   │ Stored Image
   ▼
ECS
```

ECR is therefore positioned between CI and CD.

The responsibility boundary is:

```text
ECR
  =
Container Image Storage

ECS
  =
Container Runtime / Deployment
```

ECR does not run the application.

ECS does not act as the container image registry.

---

# 9. Amazon ECS Deployment Architecture

The continuous-delivery portion begins after the image is available in ECR.

```text
Amazon ECR
     │
     │ Container Image
     ▼
Jenkins Deployment Stage
     │
     │ ECS Service Update
     ▼
Amazon ECS
     │
     ▼
ECS Service
     │
     ▼
Running Container
```

The demonstrated deployment updates the ECS service using a forced new deployment.

Conceptually:

```text
New Image
   ↓
ECR
   ↓
ECS Service Update
   ↓
New Deployment
   ↓
Running Container
```

This is the demonstrated deployment model for the project.

---

# 10. CI/CD Boundary

The overall architecture can therefore be divided into:

```text
                 CONTINUOUS INTEGRATION

GitHub
   ↓
Jenkins
   ↓
Build
   ↓
Test
   ↓
Code Quality
   ↓
Quality Gate
   ↓
Nexus
   ↓
Docker
   ↓
ECR


                 CONTINUOUS DELIVERY

ECR
 ↓
Jenkins
 ↓
ECS
 ↓
Running Container
```

The important transition is:

```text
Application Artifact
        ↓
Container Image
        ↓
Container Registry
        ↓
Container Runtime
```

---

# 11. Trigger Architecture

The pipeline does not depend exclusively on manual execution.

The project demonstrates multiple ways to start Jenkins execution.

```text
                         ┌─────────────────┐
                         │     GitHub      │
                         └────────┬────────┘
                                  │
                              Webhook
                                  │
                                  ▼
┌──────────────┐            ┌───────────┐
│ Poll SCM     │───────────►│           │
└──────────────┘            │  Jenkins  │
                            │           │
┌──────────────┐            │           │
│ Periodic     │───────────►│           │
│ Schedule      │            └─────┬─────┘
└──────────────┘                  │
                                  │
┌──────────────┐                  │
│ Remote API   │──────────────────┘
└──────────────┘
```

The mechanisms serve different purposes:

### GitHub Webhook

GitHub notifies Jenkins when the configured repository event occurs.

```text
GitHub
   │
   │ HTTP POST
   ▼
Jenkins /github-webhook/
```

This is event-driven.

### Poll SCM

Jenkins periodically checks the repository for changes.

```text
Jenkins
   │
   │ Periodic Check
   ▼
GitHub
```

### Build Periodically

Jenkins executes according to a schedule without requiring a source-code change.

```text
Cron Schedule
      ↓
Jenkins
      ↓
Build
```

### Remote Trigger

An external system can trigger Jenkins through an HTTP request.

```text
External System / Script
          ↓
       HTTP API
          ↓
        Jenkins
```

---

# 12. Notification Architecture

Slack provides the feedback path from Jenkins to the team.

```text
Pipeline
   ↓
Build Result
   ↓
Jenkins Post Action
   ↓
Slack
```

The notification contains information such as:

- Build result
- Job name
- Build number
- Jenkins build URL

The conceptual feedback loop is:

```text
Code Change
    ↓
Pipeline
    ↓
Build / Test / Analysis
    ↓
Result
    ↓
Slack Notification
```

This makes the pipeline outcome visible without requiring someone to continuously monitor Jenkins.

---

# 13. Jenkins Security Architecture

Security is divided into two separate concerns:

```text
Authentication
      ↓
Who are you?

Authorization
      ↓
What are you allowed to do?
```

The project demonstrates Jenkins' own user database and multiple authorization strategies.

---

## 13.1 Authentication

The Jenkins user database provides the identity layer.

```text
User
 ↓
Jenkins Login
 ↓
Authenticated Identity
```

The practical also covers other authentication mechanisms conceptually, including LDAP, but LDAP is not part of the demonstrated implementation.

---

## 13.2 Authorization

Authorization determines the permissions associated with an authenticated user.

The project explores:

```text
Matrix-Based Security
        ↓
Project-Based Matrix Security
        ↓
Role-Based Authorization
```

These approaches provide progressively different levels of permission management.

---

## 13.3 Project-Based Security

Project-based authorization provides a job-level boundary.

Conceptually:

```text
User
  │
  ├── Job A → Build
  │
  └── Job B → No Access
```

This is different from global matrix authorization, where permissions apply more broadly across the Jenkins instance.

---

## 13.4 Role-Based Security

Role-based authorization groups permissions into reusable roles.

```text
Role
  ↓
Permission Set
  ↓
Users
```

This reduces the need to configure every permission independently for every user.

---

# 14. Credentials Boundary

Credentials are required when Jenkins communicates with external systems.

Examples include credentials for:

```text
Jenkins
  ├── GitHub / Git
  ├── Nexus
  ├── AWS
  └── Other integrations
```

The architecture therefore separates:

```text
Pipeline Logic
      ≠
Secret Values
```

The Jenkinsfile references credential identifiers rather than embedding secret values directly in pipeline logic.

Secrets must not be committed to the public repository.

---

# 15. Plugin Architecture

Jenkins uses plugins to integrate with external tools and services.

The project relies on integrations for areas such as:

- Git
- Maven
- SonarQube
- Nexus
- Docker
- Slack
- AWS
- Pipeline functionality

The conceptual relationship is:

```text
                Jenkins Core
                     │
             ┌───────┼────────┐
             ▼       ▼        ▼
          Plugin   Plugin   Plugin
             │       │        │
             ▼       ▼        ▼
          Tool A   Tool B   Tool C
```

An important architectural distinction is:

```text
Jenkins Plugin
      ≠
Actual External Tool
```

For example, installing an integration plugin does not automatically mean that the corresponding external service or executable has been installed and configured.

---

# 16. Failure Propagation

The pipeline is intentionally structured as a sequence of dependent stages.

```text
Fetch Code
    ↓
  Build
    ↓
 Unit Test
    ↓
 Checkstyle
    ↓
 SonarQube
    ↓
Quality Gate
    ↓
  Nexus
    ↓
 Docker
    ↓
  ECR
    ↓
  ECS
```

A failure in an earlier required stage should prevent later stages from producing or deploying an invalid result.

For example:

```text
Quality Gate = FAIL
       ↓
Pipeline stops
       ↓
No normal artifact/container progression
```

This establishes a control flow rather than treating each stage as an unrelated command.

---

# 17. Artifact and Deployment Boundaries

The project contains several different artifact types.

```text
Java Source
    ↓
WAR Artifact
    ↓
Docker Image
```

Each has a different storage/runtime responsibility:

| Artifact | Location / System | Purpose |
|---|---|---|
| Source | GitHub | Source history |
| WAR | Nexus | Versioned application artifact |
| Docker Image | Amazon ECR | Container deployment artifact |
| Running Container | Amazon ECS | Runtime workload |

This progression is one of the central architectural ideas of the project.

---

# 18. End-to-End Architecture

Combining all major components:

```text
                                      ┌──────────────┐
                                      │   Developer  │
                                      └──────┬───────┘
                                             │
                                             │ Push
                                             ▼
                                      ┌──────────────┐
                                      │    GitHub    │
                                      └──────┬───────┘
                                             │
                                      Webhook / Trigger
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                              JENKINS                                │
│                                                                     │
│  Fetch → Build → Test → Checkstyle → SonarQube → Quality Gate      │
│                                      │                              │
│                                      ▼                              │
│                                   Nexus                              │
│                                      │                              │
│                                      ▼                              │
│                              Docker Image                            │
│                                      │                              │
│                                      ▼                              │
│                                    ECR                               │
│                                      │                              │
│                                      ▼                              │
│                                    ECS                               │
└─────────────────────────────────────────────────────────────────────┘
                                             │
                                             ▼
                                      Running Container
                                             │
                                             ▼
                                      Slack Notification
```

The supporting control-plane elements are:

```text
GitHub Webhooks / Polling / Schedule / Remote Trigger
                         │
                         ▼
                      Jenkins
                         │
                 ┌───────┴────────┐
                 ▼                ▼
          Credentials        Authorization
                 │                │
                 └───────┬────────┘
                         ▼
                      Pipeline
```

---

# 19. Major Architectural Decisions

## 19.1 Jenkins as the Orchestrator

Jenkins coordinates the complete delivery workflow while specialized tools perform their respective responsibilities.

**Reason:**

The project demonstrates integration of multiple DevOps tools through one automated workflow.

---

## 19.2 Nexus for Application Artifacts

The WAR artifact is stored in Nexus rather than relying on the Jenkins workspace as the permanent artifact location.

**Reason:**

The artifact requires centralized, versioned storage that downstream processes can retrieve.

---

## 19.3 Docker as the Deployment Unit

The application is transformed from a WAR artifact into a Docker image.

**Reason:**

The ECS deployment operates on a container image.

---

## 19.4 ECR Between CI and ECS

ECR is used as the image registry between image creation and container deployment.

```text
Jenkins
  ↓
ECR
  ↓
ECS
```

**Reason:**

The registry and runtime have separate responsibilities.

---

## 19.5 Quality Gate Before Artifact/Deployment Progression

The Quality Gate is positioned before the downstream artifact/container deployment flow.

**Reason:**

The pipeline should establish an acceptable quality result before continuing.

---

## 19.6 Build-Specific Container Tags

The demonstrated Docker pipeline uses Jenkins build information when tagging images.

**Reason:**

A build-specific tag creates traceability between the Jenkins build and the generated container image.

The project also maintains `latest`, but this introduces a limitation discussed in the future-work documentation.

---

# 20. Architectural Boundaries

The architecture intentionally does not claim:

- VProfile application development
- Terraform-based infrastructure provisioning
- Kubernetes deployment
- GitOps
- Enterprise Jenkins high availability
- Enterprise LDAP implementation
- Production-grade secrets management
- Immutable ECS task-definition deployment
- Guaranteed zero-downtime deployment

These are outside the demonstrated architectural scope.

For the detailed boundaries and future evolution:

**[Limitations & Future Work →](limitations-and-future-work.md)**

---

# 21. Future Architectural Evolution

The current architecture provides a foundation for further evolution.

### Current

```text
GitHub
   ↓
Jenkins
   ↓
Build / Test / Quality
   ↓
Nexus
   ↓
Docker
   ↓
ECR
   ↓
ECS
```

### Potential evolution

```text
GitHub
   ↓
Jenkins
   ↓
Automated CI
   ↓
Security / Quality Gates
   ↓
Immutable Container Image
   ↓
ECR
   ↓
Versioned ECS Task Definition
   ↓
Automated ECS Deployment
   ↓
Rollback
```

Further evolution could introduce:

```text
Infrastructure as Code
        ↓
Terraform
        ↓
Cloud Infrastructure
```

and later:

```text
Container Platform
        ↓
Kubernetes / EKS
        ↓
GitOps
```

These are future architectural directions and are not claimed as implemented capabilities of this project.

---

## 22. Architecture Summary

The project's architecture can be compressed into one mental model:

```text
                    SOURCE
                      │
                      ▼
                    GitHub
                      │
                 Trigger Event
                      │
                      ▼
                  JENKINS
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
        Build       Quality      Artifact
        /Test       Analysis     Storage
          │           │            │
          └───────────┴────────────┘
                      │
                      ▼
                   Docker
                      │
                      ▼
                     ECR
                      │
                      ▼
                     ECS
                      │
                      ▼
              Running Container
                      │
                      ▼
                  Notification
```

The central architectural idea is:

> **Jenkins orchestrates the software delivery lifecycle while specialized systems handle source control, build/test, quality analysis, artifact storage, container registry, and runtime deployment.**

This architecture represents the **DevOps engineering performed around the existing application workload**, rather than ownership of the application itself.

---

[← Back to README](../README.md) | [Implementation](implementation.md) | [Validation](validation.md) | [Limitations & Future Work](limitations-and-future-work.md)
