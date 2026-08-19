# Jenkins CI/CD Platform — Implementation

[← Back to README](../README.md) | [Architecture](architecture.md) | [Validation](validation.md) | [Limitations & Future Work](limitations-and-future-work.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/cf7af122-7803-4541-862c-03d657d3028d" />

---

## 1. Implementation Overview

This document records how the Jenkins CI/CD platform was assembled and configured around the existing VProfile application workload.

The implementation progressed from Jenkins infrastructure and integrations into a complete Pipeline as Code workflow:

```text
Jenkins Infrastructure
        ↓
External Tooling
        ↓
Jenkins Plugins
        ↓
Credentials
        ↓
Pipeline Configuration
        ↓
Build / Test
        ↓
Code Quality
        ↓
Artifact Publishing
        ↓
Docker Image
        ↓
Amazon ECR
        ↓
Amazon ECS
        ↓
Triggers / Notifications / Security
```

The implementation is intentionally documented as an engineering sequence rather than as a reproduction of the course material.

---

# 2. Implementation Boundary

The implementation work represented here covers the DevOps environment and delivery workflow around the existing application.

### Implemented / configured

- Jenkins
- Git/GitHub integration
- Maven build and test execution
- Checkstyle
- SonarQube integration
- SonarQube Quality Gate
- Nexus artifact publishing
- Docker Engine on the Jenkins server
- Docker image creation
- Amazon ECR publishing
- Amazon ECS deployment
- Jenkins credentials
- GitHub webhook triggering
- Poll SCM
- Scheduled builds
- Remote triggering
- Slack notifications
- Jenkins authentication and authorization

### Outside the implementation boundary

- Development of the VProfile application
- VProfile business logic
- VProfile functional architecture
- Terraform infrastructure provisioning
- Kubernetes deployment
- GitOps
- Enterprise LDAP implementation
- Enterprise Jenkins high availability

The VProfile application is treated as the workload consumed by the delivery platform.

---

# 3. Implementation Sequence

The overall implementation followed this sequence:

```text
1. Jenkins Server
       ↓
2. Nexus Server
       ↓
3. SonarQube Server
       ↓
4. AWS Security Groups
       ↓
5. Jenkins Plugins
       ↓
6. Jenkins ↔ Nexus Integration
       ↓
7. Jenkins ↔ SonarQube Integration
       ↓
8. Jenkins Pipeline
       ↓
9. Slack Notifications
       ↓
10. Automated Build Triggers
       ↓
11. Docker Engine + ECR
       ↓
12. Docker CI Pipeline
       ↓
13. ECS Prerequisites
       ↓
14. ECS Deployment Stage
       ↓
15. Jenkins Security
```

This sequence reflects the practical progression from infrastructure to integrations and finally to automated delivery.

---

# 4. Jenkins Infrastructure

## 4.1 Jenkins Server

Jenkins was hosted on an AWS EC2 instance.

The Jenkins server acts as the central automation environment for the project.

Conceptually:

```text
AWS
 │
 └── EC2
      │
      └── Jenkins
```

The initial Jenkins environment provides the execution platform on which the pipeline and its supporting tools run.

---

## 4.2 Installation Approach

The practical uses AWS EC2 instance initialization and user-data scripts as part of the infrastructure setup.

The purpose is to automate initial server configuration rather than requiring every installation step to be performed manually after the instance starts.

The same general approach is used for the Jenkins, Nexus and SonarQube environments in the CI setup.

---

# 5. Nexus Setup

Nexus was deployed as a separate service used for application artifact storage.

Conceptually:

```text
AWS EC2
   │
   └── Nexus
        │
        └── vprofile-repo
```

The purpose of Nexus is to provide centralized storage for the generated application artifact.

The Jenkins workspace is therefore not treated as the permanent artifact repository.

---

## 5.1 Jenkins → Nexus Integration

Jenkins credentials were configured for Nexus authentication.

The pipeline references the stored credential rather than embedding the username and password in the pipeline logic.

The integration boundary is:

```text
Jenkins
   │
   │ Credentials
   ▼
Nexus
   │
   ▼
WAR Artifact
```

The pipeline later publishes the generated WAR file to the configured Nexus repository.

---

# 6. SonarQube Setup

SonarQube was deployed as the code-quality analysis service.

```text
AWS EC2
   │
   └── SonarQube
```

SonarQube is positioned between the application build/test stages and downstream artifact progression.

The integration requires more configuration than the Nexus integration because Jenkins must be able to communicate with the SonarQube server and receive the Quality Gate result.

---

## 6.1 Jenkins → SonarQube Integration

The integration includes:

- SonarQube server configuration
- Authentication/token configuration
- Jenkins-side integration
- Quality Gate communication
- Webhook support where required by the demonstrated configuration

The conceptual flow is:

```text
Jenkins
   │
   │ Source / Analysis Request
   ▼
SonarQube
   │
   │ Analysis Result
   ▼
Quality Gate
   │
   ▼
Jenkins
```

---

# 7. Network Configuration

The Jenkins, Nexus and SonarQube services require network connectivity between their respective AWS environments.

The practical therefore includes security-group configuration.

Important communication paths include:

```text
Jenkins ───────► Nexus
        port 8081

Jenkins ───────► SonarQube
        port 9000
```

The exact network rules should be restricted to the required sources and ports rather than broadly exposing every service.

A common integration failure is a connection timeout caused by an incorrect AWS security-group rule.

For troubleshooting:

```text
Integration Failure
       ↓
Check DNS / Address
       ↓
Check Service Availability
       ↓
Check Security Group
       ↓
Check Port
       ↓
Check Credentials
       ↓
Check Jenkins Configuration
```

---

# 8. Jenkins Plugin Configuration

Jenkins was extended through plugins required for the pipeline integrations.

The CI implementation specifically identifies plugins for:

- Git
- Maven
- SonarQube
- Nexus

The Docker/ECR portion additionally requires Docker and AWS-related Jenkins integrations.

The plugin layer can be viewed as:

```text
Jenkins
 │
 ├── Git integration
 ├── Maven integration
 ├── SonarQube integration
 ├── Nexus integration
 ├── Docker integration
 ├── AWS integration
 └── Slack integration
```

A plugin provides Jenkins integration capability; it does not automatically install or configure the external service or executable.

---

# 9. Jenkins Credentials

Credentials were stored in Jenkins for communication with external systems.

The practical uses credentials for systems such as:

```text
Jenkins
  ├── GitHub / Git
  ├── Nexus
  ├── SonarQube
  ├── AWS
  └── ECR
```

The important implementation principle is:

```text
Credential Secret
       ↓
Jenkins Credential Store
       ↓
Credential ID
       ↓
Pipeline
```

The pipeline references credential identifiers instead of placing secrets directly into the Jenkinsfile.

No secrets should be committed to this GitHub repository.

---

# 10. Pipeline as Code

The core automation was implemented using a Jenkins Pipeline script.

The pipeline is the glue connecting the previously configured resources:

```text
GitHub
   ↓
Jenkinsfile / Pipeline
   ├── Git
   ├── Maven
   ├── Checkstyle
   ├── SonarQube
   ├── Nexus
   ├── Docker
   ├── ECR
   └── ECS
```

The pipeline defines the execution order and determines whether downstream stages can proceed.

---

# 11. CI Pipeline Implementation

The core CI sequence is:

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
Publish to Nexus
```

The Docker extension adds:

```text
Build Docker Image
    ↓
Push to ECR
    ↓
Cleanup Local Image
```

The ECS extension adds:

```text
Deploy to ECS
```

---

# 12. Source Checkout

The pipeline retrieves the application source from GitHub.

The Docker CI implementation uses the `docker` branch.

Conceptually:

```text
GitHub
   │
   └── docker branch
          │
          ▼
       Jenkins
```

The checkout stage establishes the source revision used by the remainder of the pipeline.

---

# 13. Maven Build

The build stage uses Maven to produce the application artifact.

The demonstrated flow uses:

```bash
mvn install -DskipTests
```

for the initial build/package operation.

Conceptually:

```text
Source Code
     ↓
   Maven
     ↓
   WAR File
```

The generated application artifact becomes an input to both Nexus publishing and Docker image creation.

---

# 14. Unit Tests

Testing is performed separately through the Maven test lifecycle.

The demonstrated command is:

```bash
mvn test
```

The stage establishes whether the application's unit tests pass before the pipeline progresses.

```text
Build
  ↓
mvn test
  ↓
Pass / Fail
```

---

# 15. Checkstyle

Checkstyle is executed as a code-style analysis step.

The demonstrated Maven goal is:

```bash
mvn checkstyle:checkstyle
```

The resulting Checkstyle report is also made available to SonarQube analysis.

The flow is:

```text
Source
  ↓
Checkstyle
  ↓
Report
  ↓
SonarQube
```

---

# 16. SonarQube Analysis

The SonarQube stage performs static code analysis.

The pipeline passes information such as:

- Project key
- Project name
- Source location
- Test report
- Coverage report
- Checkstyle report

The demonstrated configuration includes:

```text
sonar.projectKey
sonar.projectName
sonar.java.binaries
sonar.junit.reportsPath
sonar.coverage.jacoco.xmlReportPaths
sonar.java.checkstyle.reportPaths
```

The conceptual flow is:

```text
Source
   ↓
SonarQube Scanner
   ↓
SonarQube Server
   ↓
Analysis Result
```

---

# 17. Quality Gate

After the SonarQube analysis completes, the pipeline waits for the Quality Gate.

Conceptually:

```text
SonarQube Analysis
       ↓
  Quality Gate
     /     \
   PASS     FAIL
    │         │
    ▼         ▼
 Continue    Abort
```

The demonstrated Jenkins implementation uses:

```text
waitForQualityGate abortPipeline: true
```

inside a timeout block.

This makes the Quality Gate an actual pipeline control point rather than merely an informational report.

---

# 18. Nexus Artifact Publishing

After successful quality validation, the generated WAR artifact is uploaded to Nexus.

The demonstrated configuration uses:

```text
Nexus Version: nexus3
Protocol: HTTP
Repository: vprofile-repo
Credential: nexuslogin
Artifact: target/vprofile-v2.war
Type: WAR
```

The version incorporates Jenkins build information and a build timestamp.

Conceptually:

```text
target/vprofile-v2.war
          │
          ▼
        Nexus
          │
          ▼
Versioned Artifact
```

The artifact repository therefore preserves build outputs independently from the Jenkins workspace.

---

# 19. Docker Prerequisites

The Docker CI implementation requires Docker to be available on the Jenkins server.

The practical specifically installs **Docker Engine**, not Docker Desktop.

The Jenkins user also needs permission to communicate with the Docker daemon.

The implementation therefore includes:

```text
Docker Engine
     ↓
Docker daemon
     ↓
Jenkins user
     ↓
Docker commands
```

The practical identifies the Docker-group permission issue:

```text
root
  ↓
docker works

jenkins
  ↓
permission denied
```

The Jenkins user is therefore added to the Docker group and the session/server is refreshed so the group membership takes effect.

---

# 20. AWS CLI

The AWS CLI is installed on the Jenkins server for AWS interaction.

The Docker CI stage itself uses the Jenkins Docker/AWS integrations for ECR operations, while the AWS CLI is required for the later ECS deployment operation.

The relationship is:

```text
Jenkins
  │
  ├── Docker / ECR integration
  │
  └── AWS CLI
          │
          └── ECS deployment
```

---

# 21. Amazon ECR Prerequisites

An ECR repository is created for the application image.

The demonstrated repository is:

```text
vprofileappimg
```

AWS credentials are stored in Jenkins using the demonstrated credential identifier:

```text
awscreds
```

The Docker/ECR implementation also requires the relevant Jenkins plugins for AWS and Docker integration.

Conceptually:

```text
Jenkins
   │
   │ AWS Credentials
   ▼
Amazon ECR
   │
   └── vprofileappimg
```

---

# 22. Docker Image Build

The pipeline builds the application image using the Dockerfile supplied for the workload.

The demonstrated implementation uses a multistage Dockerfile located under:

```text
Docker-files/app/multistage/
```

The Jenkins pipeline creates the image through the Docker Pipeline integration.

Conceptually:

```text
WAR Artifact
     ↓
Dockerfile
     ↓
docker.build()
     ↓
Docker Image
```

The image is tagged using the Jenkins build number.

Example:

```text
vprofileappimg:42
```

---

# 23. Docker Image Publishing

The generated image is pushed to Amazon ECR.

The pipeline uses the Docker registry integration and pushes:

```text
<build-number>
latest
```

For example:

```text
vprofileappimg:42
vprofileappimg:latest
```

The build-number tag provides a relationship between the Jenkins build and the image.

The `latest` tag provides a moving reference to the most recent image.

---

# 24. Docker Cleanup

After the image is pushed to ECR, the local image is removed from the Jenkins host.

The purpose is to prevent repeated builds from consuming the Jenkins server's local disk space.

Conceptually:

```text
Build Image
     ↓
Push to ECR
     ↓
Image Stored Remotely
     ↓
Remove Local Image
```

The cleanup step is particularly relevant for CI servers because repeated Docker builds can consume substantial local storage.

---

# 25. ECS Prerequisites

Before adding the ECS deployment stage, the AWS environment requires:

- ECS cluster
- ECS task definition
- ECS service
- Appropriate IAM permissions
- Network/security-group configuration
- ECR image availability

Conceptually:

```text
ECR
 │
 └── Container Image

ECS
 ├── Cluster
 ├── Task Definition
 └── Service
```

The ECS service manages the running workload.

---

# 26. ECS Deployment Stage

The final pipeline stage updates the ECS service.

The demonstrated implementation uses:

```text
withAWS(
    credentials: 'awscreds',
    region: 'us-east-1'
)
```

and executes:

```bash
aws ecs update-service \
  --cluster ${cluster} \
  --service ${service} \
  --force-new-deployment
```

The deployment flow is:

```text
Docker Image
     ↓
Amazon ECR
     ↓
Jenkins
     ↓
AWS CLI
     ↓
ECS Service Update
     ↓
Force New Deployment
     ↓
Running Container
```

The demonstrated approach relies on the ECS service pulling the current image reference.

This is documented as the implementation used by the practical, not as an immutable deployment strategy.

---

# 27. Build Triggers

The implementation supports several Jenkins execution triggers.

## 27.1 GitHub Webhook

GitHub is configured with a webhook pointing to:

```text
/github-webhook/
```

The demonstrated webhook is configured for the push event.

The Jenkins job enables:

```text
GitHub hook trigger for GITScm polling
```

The flow becomes:

```text
git push
   ↓
GitHub
   ↓
Webhook
   ↓
Jenkins
   ↓
Pipeline
```

---

## 27.2 Poll SCM

Jenkins can periodically inspect the configured source repository.

```text
Jenkins
   ↓
Poll SCM
   ↓
Repository Change?
   ↓
Build
```

This is polling-based rather than event-driven.

---

## 27.3 Scheduled Builds

Jenkins can also execute jobs according to a schedule.

The practical introduces cron-style Jenkins scheduling for periodic execution.

```text
Schedule
   ↓
Jenkins
   ↓
Pipeline
```

---

## 27.4 Remote Trigger

The project also demonstrates triggering Jenkins remotely through an HTTP request.

Conceptually:

```text
External Client
      ↓
Jenkins HTTP Endpoint
      ↓
Job
      ↓
Pipeline
```

This can be used when another system needs to initiate Jenkins execution.

---

# 28. Slack Notifications

Slack notifications were added as the feedback mechanism for pipeline results.

The Slack integration uses the Jenkins Slack Notification plugin.

The notification includes information such as:

```text
Build Result
Job Name
Build Number
Build URL
```

The pipeline result is mapped into a success/failure notification.

Conceptually:

```text
Pipeline
   ↓
Build Result
   ↓
Slack Notification
```

The implementation therefore closes the feedback loop:

```text
Code Change
     ↓
Pipeline
     ↓
Result
     ↓
Developer / Team Notification
```

---

# 29. Jenkins Authentication

The practical distinguishes authentication from authorization.

Authentication answers:

> Who is the user?

The demonstrated Jenkins security configuration uses the Jenkins user database.

```text
User
 ↓
Jenkins Login
 ↓
Authenticated User
```

The material also discusses LDAP as an alternative authentication mechanism, but LDAP is not represented as an implemented capability of this portfolio project.

---

# 30. Jenkins Authorization

Authorization answers:

> What can the authenticated user do?

The practical covers several Jenkins authorization models:

```text
Matrix-Based Security
        ↓
Project-Based Matrix Security
        ↓
Role-Based Authorization
```

These provide different levels of permission control.

---

## 30.1 Matrix-Based Security

Matrix-based security provides permissions through a permission matrix.

Conceptually:

```text
              Read   Build   Configure
Admin          ✓       ✓        ✓
Developer      ✓       ✓        ✗
```

---

## 30.2 Project-Based Matrix Security

Project-based matrix security adds job-level permission boundaries.

Conceptually:

```text
Developer
   │
   ├── CI Job       → Build
   │
   └── Admin Job    → No Access
```

This limits permissions to specific projects/jobs.

---

## 30.3 Role-Based Authorization

Role-based authorization groups permissions into named roles.

```text
Role
  ↓
Permissions
  ↓
Users
```

This provides a more reusable permission model when the Jenkins environment grows.

---

# 31. Important Troubleshooting Areas

The practical contains several troubleshooting patterns that are valuable to preserve as engineering memory.

---

## 31.1 Jenkins ↔ Nexus Connectivity

If Jenkins cannot upload to Nexus:

```text
Check:
1. Nexus availability
2. Nexus URL
3. Security group
4. Port 8081
5. Jenkins credentials
6. Nexus repository
```

---

## 31.2 Jenkins ↔ SonarQube Connectivity

If SonarQube analysis cannot connect:

```text
Check:
1. SonarQube availability
2. Server URL
3. Port 9000
4. Security group
5. Authentication token
6. Jenkins SonarQube configuration
```

A timeout strongly suggests a networking/security-group issue.

---

## 31.3 Jenkins ↔ Docker Permission

If Docker works as root but not as Jenkins:

```text
root
  ↓
Docker works

jenkins
  ↓
Permission denied
```

The Docker group membership of the Jenkins user must be checked.

---

## 31.4 Docker Disk Consumption

If repeated Docker builds consume Jenkins disk space:

```text
Build Image
     ↓
Push Image
     ↓
Keep Local Image
     ↓
Disk Usage Grows
```

The implemented cleanup stage removes the local image after the push.

---

## 31.5 GitHub Webhook Failure

If the GitHub webhook fails:

```text
Check:
1. Webhook URL
2. /github-webhook/ path
3. Content type
4. GitHub event selection
5. Jenkins trigger configuration
6. Jenkins accessibility
7. AWS security group
8. Port 8080
```

GitHub's webhook delivery history can be used to inspect the request and Jenkins response.

---

## 31.6 ECS Deployment Failure

If ECS deployment fails:

```text
Check:
1. ECR image exists
2. ECS cluster
3. ECS service
4. Task definition
5. IAM permissions
6. AWS region
7. ECS networking
8. Container configuration
9. Image reference
```

The Jenkins deployment command itself should also be checked for the correct cluster and service variables.

---

# 32. Implementation Artifacts

The public repository should not automatically include every artifact used during implementation.

The important implementation artifacts are:

| Artifact | Repository Treatment |
|---|---|
| Jenkinsfile / pipeline definition | Keep or sanitize if legitimately publishable |
| Dockerfile | Keep only if personally authored or redistribution is permitted |
| Course-provided VProfile source | Exclude / verify ownership |
| Generated WAR | Exclude |
| Jenkins workspace | Exclude |
| Build logs | Selective evidence only |
| AWS credentials | Never commit |
| Jenkins secrets | Never commit |
| Nexus credentials | Never commit |
| SonarQube tokens | Never commit |
| Environment-specific IPs/IDs | Sanitize |
| Screenshots | Selective evidence |
| Architecture documentation | Keep |

---

# 33. Pipeline Implementation Summary

The demonstrated end-to-end pipeline can be compressed to:

```text
stage('Fetch Code')
        ↓
stage('Build')
        ↓
stage('Unit Test')
        ↓
stage('Checkstyle')
        ↓
stage('SonarQube Analysis')
        ↓
stage('Quality Gate')
        ↓
stage('Publish to Nexus')
        ↓
stage('Build App Image')
        ↓
stage('Upload App Image')
        ↓
stage('Deploy to ECS')
```

The Docker-specific stages add:

```text
docker.build()
      ↓
docker.push()
      ↓
docker.push('latest')
      ↓
docker cleanup
```

The ECS stage uses AWS credentials and AWS CLI to update the ECS service.

This creates the final demonstrated flow:

```text
GitHub
   ↓
Jenkins
   ↓
Maven
   ↓
Tests
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

---

# 34. Implementation Decisions Worth Remembering

## Pipeline as Code

The pipeline logic is represented as Jenkins Pipeline code rather than being entirely dependent on manual UI configuration.

## Separate Artifact Storage

Nexus stores application artifacts independently from Jenkins workspaces.

## Container-Based Delivery

The application WAR becomes part of a Docker image that is used as the deployment unit.

## ECR as the Image Handoff

ECR provides the image registry between CI and ECS deployment.

## Build Number Tagging

Jenkins build information is incorporated into image tags to provide traceability.

## Local Docker Cleanup

Images are removed after publishing to reduce Jenkins disk consumption.

## Quality Gate as a Control Point

The pipeline stops when the Quality Gate fails.

## Automated Triggering

GitHub webhooks, Poll SCM, scheduled execution and remote triggering reduce dependence on manual builds.

## Notifications

Slack provides build-result feedback.

## Credential Separation

Secrets remain in Jenkins credentials rather than pipeline source.

---

# 35. Implementation Boundary

The implementation demonstrates a functioning CI/CD workflow around the supplied application workload.

It does not demonstrate:

- Terraform provisioning
- Kubernetes/EKS deployment
- GitOps
- Immutable ECS task-definition deployment
- Enterprise LDAP
- Enterprise Jenkins HA
- Complete production secrets management
- Guaranteed zero-downtime deployment

These capabilities should remain outside the completed implementation claims.

---

# 36. Future Implementation Direction

The most direct next improvement to the demonstrated ECS deployment is to replace the moving `latest` reference and forced service redeployment with an immutable deployment flow.

### Current implementation

```text
Jenkins
   ↓
Build Image
   ↓
Push build-number + latest
   ↓
ECR
   ↓
force-new-deployment
   ↓
ECS
```

### Future implementation

```text
Jenkins
   ↓
Build Immutable Image
   ↓
Push Versioned Image
   ↓
ECR
   ↓
Create New ECS Task Definition Revision
   ↓
Reference Immutable Image
   ↓
Update ECS Service
   ↓
Deploy
   ↓
Rollback if Required
```

This would improve deployment traceability and rollback capability.

Other future implementation directions include:

- Terraform-based infrastructure provisioning
- Automated ECS infrastructure deployment
- Stronger secret-management integration
- Kubernetes/EKS deployment
- GitOps workflow
- More advanced deployment strategies
- Automated rollback
- Expanded observability

These are future capabilities and are not represented as completed work.

---

## 37. Implementation Summary

The implementation can be remembered as:

```text
PREPARE
  ↓
Jenkins + Nexus + SonarQube
  ↓
CONNECT
  ↓
Plugins + Credentials + Networking
  ↓
AUTOMATE
  ↓
Jenkins Pipeline
  ↓
VALIDATE
  ↓
Build + Test + Checkstyle + SonarQube + Quality Gate
  ↓
PACKAGE
  ↓
Nexus + Docker
  ↓
PUBLISH
  ↓
Amazon ECR
  ↓
DEPLOY
  ↓
Amazon ECS
  ↓
OBSERVE
  ↓
Slack + Jenkins Results
  ↓
TRIGGER
  ↓
Webhook / Poll / Schedule / Remote API
  ↓
SECURE
  ↓
Authentication + Authorization
```

The central implementation lesson is:

> **The Jenkins pipeline is the automation layer that connects independently configured DevOps services into one repeatable software delivery workflow.**

The project therefore demonstrates integration and automation of the delivery lifecycle around the existing VProfile workload rather than development of the workload itself.

---

[← Back to README](../README.md) | [Architecture](architecture.md) | [Validation](validation.md) | [Limitations & Future Work](limitations-and-future-work.md)
