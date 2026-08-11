# Jenkins CI/CD Platform — Docker, Amazon ECR & Amazon ECS

> A Jenkins-based CI/CD implementation that automates the journey from source code through build, testing, quality validation, artifact management, containerization, and deployment to Amazon ECS.

---

## Overview

This project demonstrates the configuration and integration of a **Jenkins CI/CD pipeline** for an existing Java application workload.

The pipeline progressively automates:

```text
Developer
    ↓
GitHub
    ↓
Jenkins
    ↓
Build
    ↓
Unit Test
    ↓
Checkstyle
    ↓
SonarQube Analysis
    ↓
Quality Gate
    ↓
Artifact Publishing
    ↓
Docker Image
    ↓
Amazon ECR
    ↓
Amazon ECS
    ↓
Running Application
```

Jenkins acts as the orchestration layer, coordinating the tools responsible for source retrieval, application builds, testing, quality analysis, artifact management, container image creation, and AWS deployment.

---

## Application Ownership Boundary

The **VProfile application is the workload used by this project**. It was not developed as part of this project.

The engineering work represented here focuses on the **DevOps and CI/CD platform around the application**, including Jenkins configuration, pipeline automation, tool integration, containerization, AWS registry/deployment integration, triggers, notifications, and Jenkins security.

Therefore, this repository does **not** claim ownership of:

- VProfile business logic
- VProfile application development
- The application's Java implementation
- The application's functional architecture

The project demonstrates how the existing application was **built, validated, packaged, containerized, and deployed through an automated delivery workflow**.

---

## Engineering Objective

The objective was to move from manually executed Jenkins jobs toward an automated, repeatable CI/CD workflow.

The implementation evolved through the following stages:

```text
Jenkins Fundamentals
        ↓
Jenkins Jobs
        ↓
Pipeline as Code
        ↓
Git Integration
        ↓
Maven Build & Test
        ↓
Code Quality
        ↓
Quality Gate
        ↓
Artifact Management
        ↓
Docker CI
        ↓
Amazon ECR
        ↓
Amazon ECS
```

Pipeline as Code provides a version-controlled representation of the CI/CD process rather than keeping the complete pipeline definition only inside the Jenkins UI.

---

## Architecture

The final demonstrated workflow can be viewed as two connected phases:

### Continuous Integration

```text
GitHub
   ↓
Jenkins
   ↓
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
Docker Image
   ↓
Amazon ECR
```

### Continuous Delivery

```text
Amazon ECR
     ↓
Jenkins Deployment Stage
     ↓
Amazon ECS
     ↓
Running Container
```

The CI pipeline builds, tests, analyzes, quality-gates, containerizes, and publishes the Docker image. The CD extension then deploys the image from ECR to ECS.

### High-Level Architecture

```text
                         ┌──────────────┐
                         │   Developer  │
                         └──────┬───────┘
                                │
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
                    │ SonarQube            │
                    │ Quality Gate          │
                    │ Docker Build         │
                    │ ECR Push             │
                    │ ECS Deployment       │
                    └──────┬─────────┬─────┘
                           │         │
                           ▼         ▼
                     ┌─────────┐ ┌─────────┐
                     │  Nexus  │ │   ECR   │
                     │ Artifact│ │Container│
                     │ Storage │ │Registry │
                     └─────────┘ └────┬────┘
                                      │
                                      ▼
                                ┌───────────┐
                                │    ECS    │
                                │  Service  │
                                └─────┬─────┘
                                      │
                                      ▼
                              Running Application
```

For the detailed architecture and component relationships:

**[Architecture →](docs/architecture.md)**

---

## My Engineering Contribution

The project demonstrates hands-on work across the Jenkins CI/CD lifecycle, including:

- Configuring Jenkins jobs and pipelines
- Moving from Freestyle concepts to Pipeline as Code
- Integrating GitHub as the source-control system
- Configuring Maven-based application builds
- Automating unit-test execution
- Integrating Checkstyle
- Integrating SonarQube
- Enforcing a SonarQube Quality Gate
- Publishing build artifacts to Nexus
- Building Docker images from the application
- Tagging Docker images using Jenkins build information
- Publishing Docker images to Amazon ECR
- Cleaning up local Docker images after publishing
- Extending CI into continuous delivery with Amazon ECS
- Configuring Jenkins build triggers
- Configuring GitHub webhook-based triggering
- Configuring Poll SCM and scheduled execution
- Configuring remote Jenkins triggering
- Configuring Slack build notifications
- Configuring Jenkins authentication and authorization

---

## Key Engineering Concepts Demonstrated

### Pipeline as Code

The project uses the Jenkins Pipeline model to represent the delivery workflow as code rather than relying exclusively on GUI configuration.

This makes the pipeline easier to understand, version, review, and reproduce.

### Build and Test Automation

Maven is used to build the Java application and execute unit tests as part of the pipeline.

### Code Quality

Checkstyle and SonarQube are integrated into the pipeline, followed by a Quality Gate that controls whether execution can continue.

### Artifact Management

Nexus provides centralized storage for application build artifacts rather than treating the Jenkins workspace as a permanent artifact repository.

### Containerization

The application is packaged into a Docker image so that the deployable unit is a container image rather than only a server-dependent WAR artifact.

### Amazon ECR

Amazon ECR acts as the Docker image registry between the CI and CD portions of the workflow.

### Amazon ECS

Amazon ECS provides the runtime environment for the containerized application.

### Automated Triggers

The project covers multiple Jenkins trigger mechanisms, including GitHub webhooks, Poll SCM, scheduled builds, and remote triggering.

### Jenkins Security

Authentication and authorization are treated as separate concerns, with Jenkins user management and authorization strategies used to control access.

---

## Docker Image Lifecycle

The containerized pipeline follows:

```text
Source Code
    ↓
Maven Build
    ↓
Application Artifact
    ↓
Docker Build
    ↓
Docker Image
    ↓
Image Tag
    ↓
Amazon ECR
    ↓
Amazon ECS
```

The demonstrated Docker pipeline uses Jenkins build information when tagging images and also removes local Docker images after pushing them to ECR to avoid unnecessary disk consumption on the Jenkins host.

---

## Validation

The project is validated at multiple levels:

- Jenkins pipeline execution
- Application build success
- Unit-test execution
- Checkstyle execution
- SonarQube analysis
- Quality Gate result
- Artifact publication
- Docker image creation
- Amazon ECR image publication
- ECS deployment
- Automated trigger execution
- Slack notification
- Jenkins authorization behavior

Detailed validation methodology and evidence mapping are documented here:

**[Validation →](docs/validation.md)**

---

## Project Boundaries

This repository represents a **practical Jenkins CI/CD implementation**, not a claim of a complete enterprise production platform.

The project does not establish:

- Development of the VProfile application
- Terraform-based infrastructure provisioning
- Kubernetes deployment
- GitOps implementation
- Enterprise LDAP integration
- Production-grade secrets management
- Enterprise Jenkins high availability
- Zero-downtime deployment guarantees
- Immutable ECS deployment through versioned task definitions

The demonstrated ECS deployment uses the ECR image workflow and `force-new-deployment` approach. A stronger future implementation would use immutable image references and new ECS task-definition revisions for improved traceability and rollback.

---

## Technologies

| Area | Technology |
|---|---|
| CI/CD Orchestration | Jenkins |
| Source Control | Git / GitHub |
| Build | Apache Maven |
| Testing | Maven Unit Tests |
| Code Quality | Checkstyle |
| Static Analysis | SonarQube |
| Quality Control | SonarQube Quality Gate |
| Artifact Repository | Nexus |
| Containerization | Docker |
| Container Registry | Amazon ECR |
| Container Runtime / Deployment | Amazon ECS |
| Cloud Platform | AWS |
| Notifications | Slack |
| Pipeline Definition | Jenkins Pipeline / Groovy |

---

## Repository Documentation

The repository uses a layered documentation structure.

### Architecture

System architecture, component relationships, traffic/data flow, boundaries, and major engineering decisions.

**[Read Architecture →](docs/architecture.md)**

### Implementation

How the Jenkins pipeline and supporting integrations were configured and assembled.

**[Read Implementation →](docs/implementation.md)**

### Validation

Validation strategy, results, and evidence mapping.

**[Read Validation →](docs/validation.md)**

### Limitations & Future Work

Current boundaries, explicitly unimplemented capabilities, and logical next iterations.

**[Read Limitations & Future Work →](docs/limitations-and-future-work.md)**

---

## Evidence

High-signal evidence from the completed environment should be placed under:

```text
evidence/screenshots/
```

Evidence should be limited to screenshots that demonstrate meaningful claims such as:

- Successful pipeline execution
- Pipeline stages
- SonarQube Quality Gate
- Nexus artifact publication
- Docker image in ECR
- ECS deployment
- GitHub webhook triggering
- Slack notification
- Jenkins authorization

Course screenshots or course material should not be presented as evidence of personal execution.

---

## Future Direction

The current implementation provides a foundation for further DevOps evolution.

Potential future improvements include:

```text
Current Jenkins CI/CD
        ↓
Immutable Image Deployment
        ↓
Versioned ECS Task Definitions
        ↓
Infrastructure as Code
        ↓
Automated Infrastructure Provisioning
        ↓
Kubernetes / EKS
        ↓
GitOps
```

These are **future improvements**, not capabilities claimed by this repository.

---

## Project Summary

This project demonstrates the practical integration of Jenkins with a broader DevOps toolchain:

```text
GitHub
   ↓
Jenkins
   ↓
Maven
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
Amazon ECR
   ↓
Amazon ECS
```

The core engineering contribution is the **automation and integration of the software delivery workflow around the existing application workload**.

The repository is intentionally focused on that engineering work rather than reproducing the learning material or claiming ownership of components that were supplied by the course.

---

## License / Source Attribution

The application workload and any course-provided resources used during the practical should not be redistributed unless the applicable ownership and licensing terms permit it.

This repository should contain only artifacts that are personally authored, legitimately publishable, appropriately attributed, or intentionally recreated as original portfolio documentation.

---

**[Architecture](docs/architecture.md) · [Implementation](docs/implementation.md) · [Validation](docs/validation.md) · [Limitations & Future Work](docs/limitations-and-future-work.md)**
