# 🚀 CI/CD for Flask App on AWS (GitHub → CodePipeline → CodeBuild → CodeDeploy → EC2)

This guide will teach you from **scratch** how to:

1. Connect **GitHub** to AWS using **OAuth**
2. Build a CI/CD pipeline with **AWS CodePipeline**, **CodeBuild**, and **CodeDeploy**
3. Deploy a Flask app to **EC2** automatically on every GitHub push.

> **Good for beginners (2025 edition)** — updated for the **latest AWS Console changes**.

---

## 📌 Prerequisites

Before you start, make sure you have:

- An **AWS account**
- A **GitHub account**
- Basic familiarity with:
  - Linux commands
  - Python / Flask

---

## 🏗 Step 1 — EC2 Instance Setup

1. **Launch EC2 instance**
   - AMI: **Amazon Linux 2**
   - Instance type: `t2.micro` (free tier eligible)
   - Security group: allow **SSH (22)**, **HTTP (80)**, and your app port (e.g., **8000**)
   - Attach **IAM Role** with policy:  
     - `AmazonEC2RoleforAWSCodeDeploy` (or custom: CodeDeploy + S3 read access)

2. **Install CodeDeploy agent**
   ```bash
   sudo yum update -y
   sudo yum install -y ruby wget
   cd /home/ec2-user
   wget https://aws-codedeploy-<region>.s3.<region>.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   sudo service codedeploy-agent start
   sudo systemctl enable codedeploy-agent

Why? CodeDeploy needs this agent to receive and run your deployment instructions.

⸻

🔑 Step 2 — GitHub Connection (2025 Console)
	1.	Go to: AWS Console → Developer Tools → Settings → Connections
	2.	Click Create connection
	3.	Select GitHub as provider
	4.	Choose GitHub OAuth
	•	Click Connect to GitHub
	•	Authorize AWS to access your repos
	5.	Give your connection a name (e.g., github-flask-connection)
	6.	Save — you will use this in CodePipeline Source stage.

⸻

🗂 Step 3 — Project Structure

Your repo should look like this:

		.
├── src/
│   ├── app.py                 # Flask app
│   └── requirements.txt       # Flask + Gunicorn
│
├── scripts/
│   ├── before_install.sh      # stop old app
│   ├── after_install.sh       # install deps, venv
│   ├── start.sh               # start app via systemd
│   └── health_check.sh        # curl /health
│
├── appspec.yml                # CodeDeploy config
├── buildspec.yml              # CodeBuild config
└── README.md


⸻

📜 Step 4 — appspec.yml

CodeDeploy uses appspec.yml to know where to copy files and what scripts to run.

version: 0.0
os: linux
files:
  - source: src/
    destination: /opt/myapp/src
permissions:
  - object: /opt/myapp/src
    pattern: '**'
    owner: ec2-user
    group: ec2-user
hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      runas: root
  ApplicationStart:
    - location: scripts/start.sh
      runas: root
  ValidateService:
    - location: scripts/health_check.sh
      runas: root


⸻

📜 Step 5 — buildspec.yml

CodeBuild runs this to package your artifact.

version: 0.2
phases:
  install:
    commands:
      - pip install -r src/requirements.txt
  build:
    commands:
      - mkdir -p bundle
      - cp -r src bundle/src
      - cp appspec.yml bundle/
      - cp -r scripts bundle/scripts
artifacts:
  base-directory: bundle
  files:
    - '**/*'
  discard-paths: no

Tip: This ensures appspec.yml is at the root of the ZIP — required by CodeDeploy.

⸻

⚙ Step 6 — CodeDeploy Application + Deployment Group
	1.	Go to AWS Console → CodeDeploy → Applications → Create application
	•	Compute platform: EC2/On-Premises
	2.	Create Deployment Group
	•	Select your EC2 instance via tag
	•	Choose deployment settings: AllAtOnce (for single instance)
	•	Use the IAM Role you attached to EC2 earlier

⸻

🔄 Step 7 — CodePipeline Setup
	1.	Go to CodePipeline → Create pipeline
	2.	Source stage
	•	Provider: GitHub
	•	Connection: select the one you created in Step 2
	•	Repo: choose your Flask repo
	•	Branch: main
	3.	Build stage
	•	Provider: AWS CodeBuild
	•	Create new build project:
	•	Environment: Managed image → Ubuntu
	•	Runtime: Standard
	•	Artifacts: CodePipeline
	•	Buildspec: buildspec.yml in repo
	4.	Deploy stage
	•	Provider: AWS CodeDeploy
	•	Application: select from Step 6
	•	Deployment Group: select from Step 6

⸻

📈 Step 8 — Workflow Diagram

flowchart LR
  A[Push code to GitHub] --> B[CodePipeline: Source]
  B --> C[CodeBuild: buildspec.yml]
  C --> D[Artifact to S3 (managed by CodePipeline)]
  D --> E[CodeDeploy: appspec.yml + scripts]
  E --> F[EC2: codedeploy-agent executes scripts]
  F --> G[Flask app running on :8000]


⸻

✅ Step 9 — Test It
	•	Push a change to your GitHub repo
	•	CodePipeline will:
	1.	Pull the latest commit
	2.	Run CodeBuild to package the app
	3.	Deploy to EC2 via CodeDeploy
	•	Visit your EC2 Public DNS: http://<ec2-public-ip>:8000

⸻

🛠 Troubleshooting

Problem	Likely Cause	Fix
AppSpec file not found	appspec.yml not at artifact root	Fix buildspec.yml
CodeDeploy stuck on ValidateService	Health check failing	Update health_check.sh
CodeBuild fails to install deps	Wrong path to requirements.txt	Check script paths
AccessDenied for GitHub connection	Connection not authorized	Reconnect GitHub in Developer Tools


⸻

🎯 Next Steps
	•	Add Blue/Green deployments with an ALB
	•	Use GitHub Actions for CI + CodeDeploy for CD
	•	Add automated tests in CodeBuild

⸻

📚 References
	•	AWS CodeDeploy User Guide
	•	AWS CodePipeline User Guide
	•	GitHub OIDC with AWS

---

