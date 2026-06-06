# codealpha-jenkins-remoting

**QuickCart API — Distributed Build System with Jenkins**

> Part of the QuickCart DevOps Pipeline Series | CodeAlpha DevOps Internship — Task 2

---

## Business Context

After standardising the build process with Gradle, QuickCart faced a new problem:

> *"The single build server was overwhelmed. Every developer pushing code triggered a build. Builds were queuing. Developers were waiting. The team was slowing down."*

This repository solves that problem by introducing a **Jenkins Master + Agent architecture** — a distributed build system where one server manages the work and separate worker machines execute the builds.

---

## What This Repository Demonstrates

| Concern | Without This | With This |
|---|---|---|
| Build server load | Single server handles everything | Master delegates to Agents |
| Scalability | Adding developers breaks the system | Add more Agents as team grows |
| Build isolation | Builds interfere with each other | Each Agent runs independently |
| Visibility | No central record of builds | Jenkins dashboard tracks every build |
| Automation | Developers run builds manually | Every code push triggers a build |

---

## Architecture

```
Developer pushes code to GitHub
            │
            ▼
    Jenkins Master (Port 8080)
    ─────────────────────────
    │  Manages the pipeline  │
    │  Assigns build jobs     │
    │  Shows dashboard UI     │
    └────────────┬────────────┘
                 │ delegates work via Port 50000
                 ▼
    Jenkins Agent (quickcart-agent)
    ─────────────────────────────
    │  Receives build jobs        │
    │  Runs gradle clean build    │
    │  Produces the JAR artifact  │
    └─────────────────────────────┘
```

**Non-technical analogy:**
Think of a restaurant kitchen. The Head Chef (Master) receives all the orders, decides what gets cooked and when, and assigns tasks. The Line Cooks (Agents) do the actual cooking. If the restaurant gets busier, you hire more Line Cooks — not more Head Chefs.

---

## Technology Stack

| Technology | Version | Purpose |
|---|---|---|
| Jenkins | LTS | CI/CD automation server |
| Docker | Latest | Container runtime |
| Docker Compose | v3.8 | Multi-container orchestration |
| Java | 17 | Required by the QuickCart application |
| Gradle | 9.5.1 | Build tool running inside the Agent |

---

## Project Structure

```
codealpha-jenkins-remoting/
│
├── docker-compose.yml      # Defines Master + Agent containers and network
├── Dockerfile.agent        # Builds Agent image with Java 17 and Gradle
├── Jenkinsfile             # Pipeline definition (Checkout → Build → Test → Package)
└── README.md               # Documentation
```

---

## Prerequisites

| Requirement | Download |
|---|---|
| Docker Desktop | https://www.docker.com/products/docker-desktop |

Verify Docker is running:
```bash
docker --version
docker-compose --version
```

---

## Setup Instructions

### Step 1 — Clone this repository

```bash
git clone https://github.com/Mexcelcloud/codealpha-jenkins-remoting.git
cd codealpha-jenkins-remoting
```

### Step 2 — Start Jenkins Master and Agent

```bash
docker-compose up -d
```

This command:
- Pulls the Jenkins Master image from Docker Hub
- Builds the Agent image using Dockerfile.agent
- Creates a private network between them
- Starts both containers in the background

Verify both containers are running:
```bash
docker ps
```

Expected output:
```
CONTAINER ID   IMAGE             STATUS         NAMES
xxxxxxxxxxxx   jenkins/jenkins   Up 2 minutes   jenkins-master
xxxxxxxxxxxx   jenkins-agent     Up 2 minutes   jenkins-agent
```

### Step 3 — Unlock Jenkins Master

Open your browser and go to:
```
http://localhost:8080
```

Jenkins will ask for an initial admin password. Retrieve it by running:
```bash
docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the password, paste it into the browser, and click **Continue**.

### Step 4 — Install Suggested Plugins

On the next screen click **Install suggested plugins** and wait for installation to complete.

### Step 5 — Create Admin User

Fill in your details and create the admin account.

### Step 6 — Connect the Agent

1. Go to **Manage Jenkins → Nodes → New Node**
2. Node name: `quickcart-agent`
3. Select **Permanent Agent** → click **Create**
4. Fill in the following:
   - Remote root directory: `/home/jenkins/agent`
   - Labels: `quickcart-agent`
   - Launch method: **Launch agent by connecting it to the controller**
5. Click **Save**
6. Copy the **secret token** shown on the agent page

### Step 7 — Start the Agent with the Secret

Stop the current agent container:
```bash
docker-compose down
```

Create a `.env` file in the project folder:
```bash
echo "JENKINS_SECRET=your_secret_token_here" > .env
```

Replace `your_secret_token_here` with the token you copied from Jenkins.

Restart the containers:
```bash
docker-compose up -d
```

The Agent will now connect automatically to the Master.

### Step 8 — Create the Pipeline

1. Go to Jenkins dashboard → **New Item**
2. Name: `quickcart-pipeline`
3. Select **Pipeline** → click **OK**
4. Under **Pipeline** section select **Pipeline script from SCM**
5. SCM: **Git**
6. Repository URL: `https://github.com/Mexcelcloud/codealpha-jenkins-remoting.git`
7. Branch: `*/main`
8. Script path: `Jenkinsfile`
9. Click **Save**

### Step 9 — Run the Pipeline

Click **Build Now** on the pipeline page.

Jenkins will:
1. Check out the QuickCart source code from GitHub
2. Run `gradle clean build` on the Agent
3. Execute all automated tests
4. Verify the JAR artifact was produced

---

## Understanding the Jenkinsfile

```groovy
pipeline {
    agent { label 'quickcart-agent' }   // Run on our Agent, not the Master
```
> This tells Jenkins: do not run this build on the Master. Send it to the Agent labelled `quickcart-agent`. The Master manages; the Agent works.

```groovy
    stages {
        stage('Checkout') { ... }   // Pull source code from GitHub
        stage('Build')    { ... }   // Run gradle clean build
        stage('Test')     { ... }   // Run automated tests
        stage('Package')  { ... }   // Verify JAR was produced
    }
```
> Each stage is a visible step in the Jenkins dashboard. If any stage fails, the pipeline stops and reports exactly where the failure occurred.

---

## How the Build Works End to End

```
Developer pushes code to GitHub
            │
            ▼
Jenkins Master detects the change
            │
            ▼
Master sends build job to quickcart-agent
            │
            ▼
Agent runs Jenkinsfile stages:
  [Checkout] → pulls code from GitHub
  [Build]    → gradle clean build
  [Test]     → gradle test
  [Package]  → verifies quickcart-api-1.0.0.jar exists
            │
            ▼
Build result reported back to Master dashboard
```

---

## Where This Fits in the QuickCart Pipeline

```
Developer writes code
        │
        ▼
Gradle builds the artifact         ← Done: codealpha-gradle-build
        │
        ▼
[ This Repository ]
  Jenkins Master + Agent
  → Automates the build
  → Runs tests
  → Produces verified artifact
        │
        ▼
Docker packages the artifact       ← Next: codealpha-docker-webserver
        │
        ▼
Azure deploys to production        ← Next: codealpha-azure-cicd
```

---

## Build Tools Landscape

Jenkins is one of several CI/CD tools that solve the same problem — automating the build, test, and delivery pipeline.

| Tool | Type | Used By |
|---|---|---|
| Jenkins | Self-hosted CI/CD | Enterprises, large teams |
| GitHub Actions | Cloud CI/CD | GitHub-hosted projects |
| GitLab CI | Cloud/Self-hosted | GitLab users |
| CircleCI | Cloud CI/CD | Startups, SaaS companies |
| Azure Pipelines | Cloud CI/CD | Microsoft ecosystem |

### Why This Matters for DevOps

The tool changes. The concept does not:

> *Detect a code change → Trigger a build → Run tests → Produce a verified artifact → Report the result*

This repository uses Jenkins — but the pipeline thinking applied here transfers directly to GitHub Actions, GitLab CI, or Azure Pipelines.

---

## Learning Outcomes

By completing this repository you have demonstrated:

- Setting up a distributed Jenkins Master + Agent architecture using Docker
- Understanding why distributed build systems exist as a business solution
- Writing a declarative Jenkinsfile pipeline with multiple stages
- Connecting a Jenkins Agent to a Master using a secret token
- Running an automated build pipeline against a real Java application
- Thinking about CI/CD automation as a business reliability problem

---

## Author

**CodeAlpha DevOps Internship**
Task 2 — Jenkins Remoting
QuickCart DevOps Pipeline Series