# Jenkins CI/CD Platform — Validation

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md) | [Limitations & Future Work](limitations-and-future-work.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/1f3a6a64-9f1f-4db5-85e1-12e61e2b3b7f" />

---

## 1. Validation Overview

This document defines how the Jenkins CI/CD platform is validated from source retrieval through container deployment.

The validation strategy follows the complete delivery path:

```text
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
   ↓
Running Application
```

The demonstrated pipeline explicitly contains ten major stages:

1. Fetch code
2. Build
3. Unit test
4. Checkstyle analysis
5. SonarQube analysis
6. Quality Gate
7. Publish to Nexus
8. Build application image
9. Upload image to ECR
10. Deploy to ECS

Validation therefore does not rely on a single successful Jenkins status. Each major capability has its own validation point.

---

# 2. Validation Philosophy

The validation approach follows:

```text
Capability
    ↓
Validation Method
    ↓
Expected Result
    ↓
Evidence
    ↓
Claim Boundary
```

For example:

```text
Docker Image Publishing
        ↓
Inspect ECR repository
        ↓
Expected image/tag exists
        ↓
ECR screenshot
        ↓
Proves image publication
```

This distinction is important because:

> A successful pipeline stage proves that the stage completed; it does not automatically prove every downstream system or business outcome.

---

# 3. Validation Levels

The project is validated at five levels.

```text
Level 1 — Pipeline Execution
        ↓
Level 2 — Individual Stage Results
        ↓
Level 3 — External System State
        ↓
Level 4 — Deployment State
        ↓
Level 5 — Evidence / Interview Proof
```

---

## Level 1 — Pipeline Execution

Confirm that Jenkins completes the pipeline successfully.

Expected:

```text
Pipeline Result = SUCCESS
```

The Jenkins stage visualization should show the expected sequence.

---

## Level 2 — Individual Stage Results

Validate each major pipeline stage independently.

```text
Fetch
Build
Test
Checkstyle
SonarQube
Quality Gate
Nexus
Docker
ECR
ECS
```

A successful final pipeline should not replace stage-level reasoning.

---

## Level 3 — External System State

Verify that the external systems contain the expected outputs.

Examples:

```text
SonarQube → Analysis Result
Nexus     → WAR Artifact
ECR       → Docker Image
ECS       → Running Task
Slack     → Build Notification
```

---

## Level 4 — Deployment State

Confirm that ECS actually has a running workload after the deployment stage.

---

## Level 5 — Evidence

Capture only high-signal evidence that supports important project claims.

Recommended evidence includes:

- Jenkins successful pipeline
- Jenkins stage visualization
- SonarQube Quality Gate
- Nexus artifact
- ECR image
- ECS running service/task
- GitHub webhook
- Slack notification
- Jenkins authorization

The README already defines these as the preferred evidence categories.

---

# 4. Validation Matrix

| Capability | Validation Method | Expected Result | Evidence |
|---|---|---|---|
| GitHub integration | Run pipeline against configured repository | Source retrieved successfully | Jenkins console/stage |
| Source branch | Inspect checkout stage | Expected branch/revision fetched | Jenkins console |
| Maven build | Execute pipeline build stage | Build succeeds and WAR exists | Jenkins console |
| Unit tests | Execute Maven test stage | Tests complete successfully | Jenkins console/test result |
| Checkstyle | Execute Checkstyle stage | Checkstyle analysis completes | Jenkins console/report |
| SonarQube | Inspect SonarQube project | Analysis result available | SonarQube screenshot |
| Quality Gate | Inspect gate result | Gate passes before progression | Quality Gate screenshot |
| Nexus | Inspect repository | Versioned WAR exists | Nexus screenshot |
| Docker | Inspect Jenkins build | Image created successfully | Jenkins console |
| ECR | Inspect ECR repository | Expected image/tag exists | ECR screenshot |
| Docker cleanup | Inspect post-build output | Local image cleanup executes | Jenkins console |
| ECS | Inspect service/tasks | New deployment reaches running state | ECS screenshot |
| GitHub webhook | Inspect webhook delivery | Jenkins receives trigger | GitHub webhook screenshot |
| Poll SCM | Configure and trigger polling | Repository changes detected | Jenkins build history |
| Scheduled build | Configure schedule | Jenkins starts according to schedule | Jenkins build history |
| Remote trigger | Send authorized request | Jenkins starts target job | Jenkins build history |
| Slack | Complete/fail a build | Notification appears | Slack screenshot |
| Authentication | Log in with configured user | Correct identity authenticated | Jenkins UI |
| Authorization | Access restricted job/function | Permissions enforced | Jenkins UI |
| Credentials | Execute integration requiring credential | Authentication succeeds without exposing secret | Jenkins console/config |

---

# 5. Source-Control Validation

## Objective

Verify that Jenkins can retrieve the application source from GitHub.

The demonstrated Docker pipeline uses the `docker` branch.

### Validation

Run the Jenkins pipeline and inspect the checkout stage.

Expected:

```text
GitHub
   ↓
Checkout
   ↓
Expected branch/revision
   ↓
Workspace populated
```

### Success criteria

- Repository checkout succeeds.
- Expected branch is used.
- Jenkins identifies the source revision.
- No Git authentication or connectivity error occurs.

### Evidence

Recommended:

```text
Jenkins Console
```

showing the checkout operation and source revision.

---

# 6. Maven Build Validation

## Objective

Verify that the application can be built successfully.

### Validation

Execute the build stage.

The demonstrated build command is:

```bash
mvn install -DskipTests
```

### Expected result

```text
Maven
  ↓
Build Success
  ↓
WAR Artifact
```

### Success criteria

- Maven exits successfully.
- The application is packaged.
- The expected WAR artifact is produced.

### Evidence

Jenkins console output showing successful Maven execution.

---

# 7. Unit-Test Validation

## Objective

Verify that automated unit tests execute as part of the pipeline.

The demonstrated test command is:

```bash
mvn test
```

### Expected result

```text
Build
  ↓
Unit Tests
  ↓
PASS
```

### Success criteria

- Maven test execution completes.
- No test failure causes the pipeline to abort.
- Test reports are generated where configured.

### Evidence

Use the Jenkins test result or console output.

---

# 8. Checkstyle Validation

## Objective

Verify that Checkstyle analysis executes successfully.

The demonstrated command is:

```bash
mvn checkstyle:checkstyle
```

### Expected result

```text
Source
  ↓
Checkstyle
  ↓
Report
```

### Success criteria

- Checkstyle execution completes.
- The expected report is generated.
- The report is available for downstream SonarQube analysis.

### Evidence

Prefer the Jenkins console or the generated analysis result rather than storing large report files in the repository.

---

# 9. SonarQube Validation

## Objective

Verify that Jenkins successfully sends the analysis to SonarQube.

The pipeline combines information from source analysis, tests, coverage, and Checkstyle before the Quality Gate decision.

### Expected flow

```text
Jenkins
   ↓
SonarQube Scanner
   ↓
SonarQube Server
   ↓
Analysis
   ↓
Quality Gate
```

### Success criteria

- Analysis completes successfully.
- The expected SonarQube project is updated.
- Analysis results are available.
- The pipeline can retrieve the Quality Gate result.

### Evidence

Recommended:

**SonarQube project/result screenshot**

The screenshot should show the relevant project and analysis result without exposing credentials or internal secrets.

---

# 10. Quality Gate Validation

## Objective

Verify that the Quality Gate acts as an actual pipeline control point.

### Expected flow

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

### Positive validation

Expected:

```text
Quality Gate = PASS
       ↓
Pipeline continues
```

### Negative validation

Where safely reproducible, introduce a controlled condition that causes the gate to fail.

Expected:

```text
Quality Gate = FAIL
       ↓
Pipeline aborts
       ↓
Downstream stages do not continue normally
```

### Why both matter

A green pipeline proves:

> The success path works.

A controlled failure proves:

> The quality gate actually controls pipeline progression.

### Evidence

Best evidence:

- Quality Gate status
- Jenkins stage showing successful progression
- If performed, controlled failure showing pipeline abortion

Do not manufacture a failure screenshot if it was not actually performed.

---

# 11. Nexus Artifact Validation

## Objective

Verify that the application artifact is published to the configured Nexus repository.

The demonstrated pipeline publishes the WAR as a versioned artifact.

### Expected flow

```text
Maven
  ↓
WAR
  ↓
Jenkins
  ↓
Nexus
  ↓
Versioned Artifact
```

### Success criteria

- Nexus accepts the upload.
- The expected repository contains the artifact.
- The artifact has the expected build/version information.
- The artifact can be identified independently from the Jenkins workspace.

### Evidence

Recommended:

**Nexus repository screenshot**

Show:

- Repository
- Artifact
- Version

Avoid exposing credentials or private infrastructure information.

---

# 12. Docker Image Validation

## Objective

Verify that the application is successfully transformed into a Docker image.

### Expected flow

```text
WAR Artifact
     ↓
Docker Build
     ↓
Docker Image
```

### Success criteria

- Docker build completes successfully.
- Image is created.
- Image has the expected repository/tag.
- The image can be pushed to ECR.

### Evidence

Jenkins console output or ECR result is sufficient.

Avoid storing the image itself in GitHub.

---

# 13. Docker Tag Validation

## Objective

Verify that image tags provide build traceability.

The demonstrated implementation uses Jenkins build information for the image tag and also maintains a `latest` tag.

Conceptually:

```text
vprofileappimg:<build-number>
vprofileappimg:latest
```

### Validation

Compare:

```text
Jenkins Build Number
        ↕
Docker Image Tag
```

### Success criteria

The build-specific image tag corresponds to the Jenkins build that produced it.

### Why this matters

This creates a traceability relationship:

```text
Jenkins Build
      ↓
Docker Image
      ↓
ECR
```

The `latest` tag is a moving reference and should not be treated as a permanent version identifier.

---

# 14. Amazon ECR Validation

## Objective

Verify that the Docker image is successfully published to Amazon ECR.

### Expected flow

```text
Jenkins
   ↓
Docker Image
   ↓
Amazon ECR
```

### Success criteria

- ECR repository exists.
- Image upload succeeds.
- Expected build-specific tag exists.
- `latest` exists if that tag is part of the demonstrated implementation.
- Image metadata is visible.

### Evidence

Recommended:

**Amazon ECR repository screenshot**

The screenshot should clearly show the repository and relevant image tag.

---

# 15. Docker Cleanup Validation

## Objective

Verify that local Docker images are removed after successful publishing.

The practical explicitly adds cleanup to prevent accumulated Docker images from consuming Jenkins disk space.

### Expected flow

```text
Build Image
     ↓
Push to ECR
     ↓
Image Stored Remotely
     ↓
Local Cleanup
```

### Success criteria

- ECR push succeeds.
- Cleanup command executes afterward.
- Local image is removed.

### Evidence

Jenkins console output showing the cleanup operation.

---

# 16. Amazon ECS Validation

## Objective

Verify that the published container image is deployed through ECS.

### Expected flow

```text
ECR
 ↓
Jenkins
 ↓
ECS Service Update
 ↓
New Deployment
 ↓
Running Container
```

### Success criteria

- ECS deployment command succeeds.
- ECS service starts a new deployment.
- Desired task count is reached.
- New task becomes running.
- Container uses the expected image reference.
- Application becomes available through the configured service path.

### Evidence

Recommended:

**Amazon ECS service/task screenshot**

Show the running task/deployment state without exposing sensitive infrastructure information.

---

# 17. ECS Deployment Strategy Validation

The demonstrated deployment uses:

```text
ECR Image
   ↓
ECS Service
   ↓
force-new-deployment
```

This should be validated as the **demonstrated implementation**, not described as immutable deployment.

The stronger future approach is to use versioned ECS task definitions with immutable image references.

### Current validation claim

> The pipeline can trigger a new ECS deployment using the demonstrated service-update approach.

### Not validated as completed

- Immutable task-definition revisions
- Automated rollback
- Deployment history tied to immutable image digests
- Blue/green deployment
- Canary deployment

These belong in future work.

---

# 18. End-to-End Pipeline Validation

## Objective

Verify that all major stages operate together.

The expected complete sequence is:

```text
1. Fetch Code
      ↓
2. Build
      ↓
3. Unit Test
      ↓
4. Checkstyle
      ↓
5. SonarQube Analysis
      ↓
6. Quality Gate
      ↓
7. Publish to Nexus
      ↓
8. Build Docker Image
      ↓
9. Upload Image to ECR
      ↓
10. Deploy to ECS
```

### Success criteria

The complete pipeline should:

- Start successfully.
- Complete all required stages.
- Produce the application artifact.
- Produce the Docker image.
- Publish the image to ECR.
- Trigger ECS deployment.
- Reach the expected running state.

### Primary evidence

A successful Jenkins pipeline visualization is the highest-value single piece of evidence.

---

# 19. Trigger Validation

The project demonstrates several ways to initiate Jenkins execution.

---

## 19.1 GitHub Webhook

### Expected flow

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

### Validation

1. Push a controlled change to the configured repository/branch.
2. Inspect GitHub webhook delivery.
3. Confirm Jenkins receives the request.
4. Confirm a new Jenkins build starts.

### Success criteria

A repository change causes Jenkins execution without manually pressing **Build Now**.

### Evidence

GitHub webhook delivery plus Jenkins build history.

---

## 19.2 Poll SCM

### Validation

Configure Poll SCM and make a controlled repository change.

Expected:

```text
Repository Change
      ↓
Jenkins Poll
      ↓
Change Detected
      ↓
Build
```

### Evidence

Jenkins build history or console output.

---

## 19.3 Scheduled Build

### Validation

Configure a safe test schedule.

Expected:

```text
Schedule
   ↓
Jenkins
   ↓
Build
```

### Evidence

Jenkins build history showing the scheduled execution.

---

## 19.4 Remote Trigger

### Validation

Send an authorized request to the Jenkins job.

Expected:

```text
HTTP Request
    ↓
Jenkins
    ↓
Job
    ↓
Build
```

### Evidence

Jenkins build history.

Do not publish tokens used for the remote trigger.

---

# 20. Slack Notification Validation

## Objective

Verify that Jenkins communicates build results to Slack.

### Expected flow

```text
Pipeline
   ↓
Build Result
   ↓
Slack
```

### Success criteria

A completed build produces a Slack notification containing relevant build information.

The demonstrated notification includes information such as:

- Build result
- Job name
- Build number
- Build URL

### Positive validation

```text
Build = SUCCESS
      ↓
Slack Success Notification
```

### Negative validation

If a controlled failed build is actually performed:

```text
Build = FAILURE
      ↓
Slack Failure Notification
```

### Evidence

One representative Slack screenshot is sufficient.

Do not publish private Slack messages or unrelated conversation content.

---

# 21. Jenkins Authentication Validation

## Objective

Verify that Jenkins requires authentication for protected operations.

### Expected flow

```text
User
 ↓
Jenkins Login
 ↓
Authenticated Identity
```

### Success criteria

- Valid credentials authenticate successfully.
- Invalid credentials do not authenticate.
- Protected Jenkins functionality is inaccessible without authentication.

### Evidence

A screenshot of the relevant Jenkins security configuration may be sufficient.

Do not expose passwords, API tokens, or secrets.

---

# 22. Jenkins Authorization Validation

Authentication establishes identity.

Authorization determines what that identity can do.

### Validation model

```text
User
  ↓
Authenticated
  ↓
Permission Check
  ↓
Allowed / Denied
```

### Project-based example

```text
Developer
   │
   ├── CI Job → Build Allowed
   │
   └── Admin Job → Access Denied
```

### Success criteria

The configured permission model actually restricts access according to the intended role/project boundaries.

### Evidence

A controlled access test or Jenkins permission configuration screenshot.

---

# 23. Credential Validation

Credentials should be validated indirectly through the integrations that consume them.

Examples:

```text
Git Credential
     ↓
Git Checkout

Nexus Credential
     ↓
Artifact Upload

AWS Credential
     ↓
ECR / ECS Operations

SonarQube Credential
     ↓
Analysis
```

### Success criteria

- Integration authenticates successfully.
- Secret values are not printed.
- Pipeline source contains credential IDs rather than plaintext secrets.

### Security validation

Search the repository before publishing:

```text
Passwords
API Tokens
AWS Access Keys
Private Keys
Webhook Secrets
```

None should be present.

---

# 24. Failure-Path Validation

A reliable CI/CD pipeline must be validated not only for success but also for controlled failure behavior.

Important failure boundaries include:

```text
Build Failure
     ↓
Stop

Test Failure
     ↓
Stop

Quality Gate Failure
     ↓
Stop

Docker Build Failure
     ↓
Stop

ECR Push Failure
     ↓
Stop

ECS Deployment Failure
     ↓
Pipeline Failure / Investigation
```

The most important controlled failure is the Quality Gate because it is explicitly intended to prevent downstream progression when the quality criteria are not met.

---

# 25. Troubleshooting Validation

Validation should also confirm that common integration failures can be diagnosed using the appropriate layer.

| Failure | First Validation Area |
|---|---|
| Git checkout failure | Git credentials / repository / network |
| Maven failure | Build output / dependency resolution |
| Test failure | Test report / Maven output |
| Checkstyle failure | Checkstyle report |
| SonarQube timeout | Server availability / security group / port |
| Quality Gate failure | SonarQube analysis result |
| Nexus upload failure | Repository / credentials / connectivity |
| Docker permission failure | Jenkins user / Docker group |
| ECR push failure | AWS credentials / ECR permissions / repository |
| ECS deployment failure | IAM / cluster / service / task definition / image |
| Webhook failure | GitHub delivery / Jenkins endpoint / network |
| Slack failure | Slack credential / channel / plugin configuration |

This creates a layered troubleshooting model rather than treating every failure as a generic Jenkins problem.

---

# 26. Evidence Strategy

Evidence should be **minimal and high-signal**.

The repository should not become a screenshot dump.

Recommended evidence set:

```text
01-pipeline-success.png
02-pipeline-stages.png
03-sonarqube-quality-gate.png
04-nexus-artifact.png
05-ecr-image.png
06-ecs-deployment.png
07-github-webhook.png
08-slack-notification.png
09-jenkins-security.png
```

Only include files that actually exist and represent personal execution.

---

# 27. Evidence-to-Claim Mapping

| Project Claim | Best Evidence |
|---|---|
| Jenkins orchestrates CI/CD | Successful pipeline |
| Maven build works | Build stage |
| Tests execute | Test result |
| Checkstyle runs | Checkstyle stage/report |
| SonarQube integrated | SonarQube project |
| Quality Gate controls pipeline | Gate + Jenkins stage |
| Nexus stores artifacts | Nexus repository |
| Docker image created | Jenkins/ECR |
| ECR stores image | ECR repository |
| ECS deployment works | ECS service/task |
| GitHub triggers Jenkins | Webhook + Jenkins build |
| Slack notifications work | Slack message |
| Jenkins security configured | Authorization/security UI |
| Credentials are integrated | Successful authenticated operation |

---

# 28. Evidence Quality Rules

Every screenshot added to the repository should satisfy at least one of these conditions:

### Rule 1 — Proves a major capability

Example:

```text
ECR image exists
```

### Rule 2 — Proves an important integration

Example:

```text
GitHub → Jenkins webhook
```

### Rule 3 — Proves an important control

Example:

```text
Quality Gate
```

### Rule 4 — Supports an interview discussion

Example:

```text
ECS deployment
```

Avoid screenshots that merely show:

- Jenkins home page
- Generic AWS console pages
- Plugin search results
- Empty repositories
- Course slides
- Course screenshots
- Unrelated configuration screens

---

# 29. Personal Evidence Boundary

The learning material establishes what the practical demonstrates, but source material alone should not be treated as proof of personal execution.

Therefore:

```text
Course Material
      ↓
Defines Capability
```

while:

```text
Personal Screenshot / Console Output
      ↓
Provides Execution Evidence
```

The repository should only label evidence as personal execution when it genuinely comes from the completed environment.

Course screenshots should not be presented as personal evidence.

---

# 30. Validation Claims

The following claims are appropriate when the corresponding validation has actually been completed:

### Jenkins

> Jenkins successfully orchestrated the configured CI/CD stages.

### Maven

> The pipeline successfully built the application and executed its unit tests.

### SonarQube

> SonarQube analysis completed and its Quality Gate controlled pipeline progression.

### Nexus

> The generated application artifact was published to the configured Nexus repository.

### Docker

> The application was packaged into a Docker image.

### ECR

> The Docker image was published to Amazon ECR.

### ECS

> The pipeline triggered deployment of the container through Amazon ECS.

### Triggers

> Jenkins was configured to support automated execution through the demonstrated trigger mechanisms.

### Notifications

> Jenkins was configured to report build results through Slack.

### Security

> Jenkins authentication and authorization controls were configured and validated.

Only use these as completed project claims when the corresponding implementation and evidence are genuinely available.

---

# 31. Validation Boundaries

Successful validation of this project does **not** establish:

- Production-scale reliability
- High availability
- Disaster recovery
- Zero-downtime deployment
- Immutable deployment
- Automated rollback
- Kubernetes deployment
- Terraform infrastructure
- Enterprise LDAP
- Enterprise secrets management
- Full production observability

These are outside the demonstrated validation scope.

---

# 32. Final Validation Checklist

Before considering the project validated, confirm:

### Source

- [ ] GitHub repository accessible from Jenkins
- [ ] Expected branch checked out
- [ ] Source revision visible

### Build

- [ ] Maven build succeeds
- [ ] WAR artifact produced

### Tests

- [ ] Unit tests execute
- [ ] Tests pass

### Code Quality

- [ ] Checkstyle executes
- [ ] SonarQube analysis completes
- [ ] Quality Gate result is available
- [ ] Quality Gate controls progression

### Artifacts

- [ ] WAR published to Nexus
- [ ] Artifact version is identifiable

### Docker

- [ ] Docker image builds
- [ ] Build-specific image tag exists
- [ ] ECR push succeeds
- [ ] Local cleanup executes

### Deployment

- [ ] ECS service update succeeds
- [ ] New task/deployment appears
- [ ] Container reaches running state
- [ ] Application is reachable where applicable

### Automation

- [ ] GitHub webhook works, if claimed
- [ ] Poll SCM works, if claimed
- [ ] Scheduled execution works, if claimed
- [ ] Remote trigger works, if claimed

### Notifications

- [ ] Slack success notification works
- [ ] Slack failure notification works, if actually tested

### Security

- [ ] Authentication works
- [ ] Authorization boundaries work
- [ ] Credentials are stored securely
- [ ] No secrets exist in repository files

### Evidence

- [ ] Pipeline evidence captured
- [ ] Quality Gate evidence captured
- [ ] Nexus evidence captured
- [ ] ECR evidence captured
- [ ] ECS evidence captured
- [ ] Trigger evidence captured
- [ ] Notification evidence captured
- [ ] Security evidence captured
- [ ] Course screenshots excluded from personal evidence

---

# 33. Validation Summary

The complete validation model can be compressed to:

```text
SOURCE
  ↓
Can Jenkins fetch the code?
  ↓
BUILD
  ↓
Can Maven produce the artifact?
  ↓
TEST
  ↓
Do the tests pass?
  ↓
QUALITY
  ↓
Do Checkstyle and SonarQube execute?
  ↓
GATE
  ↓
Does the Quality Gate control progression?
  ↓
ARTIFACT
  ↓
Is the WAR stored in Nexus?
  ↓
CONTAINER
  ↓
Can Docker build the image?
  ↓
REGISTRY
  ↓
Is the image stored in ECR?
  ↓
DEPLOYMENT
  ↓
Does ECS run the new container?
  ↓
FEEDBACK
  ↓
Are results communicated through Slack?
  ↓
AUTOMATION
  ↓
Can changes trigger Jenkins automatically?
  ↓
SECURITY
  ↓
Are authentication and authorization enforced?
```

The final validation objective is therefore:

> **Demonstrate that the configured Jenkins platform can reliably move the existing application workload through the intended build, test, quality, artifact, container, registry, and deployment stages, with appropriate automation, feedback, and access controls.**

The strongest single validation artifact is a successful end-to-end Jenkins pipeline combined with external-system evidence showing the resulting ECR image and ECS deployment.

---

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md) | [Limitations & Future Work](limitations-and-future-work.md)
