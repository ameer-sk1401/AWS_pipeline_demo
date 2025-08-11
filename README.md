Perfect—here’s an updated README.md you can paste into your repo.
It includes a clean Mermaid workflow diagram (GitHub renders it automatically) and quick steps to export it as a PNG if you prefer an image file.

⸻


# Flask App CI/CD on AWS (CodePipeline + CodeBuild + CodeDeploy)

A beginner‑friendly, production‑ready CI/CD pipeline for a simple **Flask Hello World** app deployed to **Amazon EC2** using **AWS CodePipeline**, **CodeBuild**, and **CodeDeploy** with **GitHub** as the source.

> **Note (2025 console):** Create your GitHub connection at  
> **AWS Console → Developer Tools → Settings → Connections → Create connection (GitHub OAuth)**.  
> Select this connection in the **CodePipeline Source** stage.

---

## Architecture (Diagram)

```mermaid
flowchart LR
  A[Developer\nPush to GitHub] --> B[CodePipeline\nSource: GitHub Connection]
  B --> C[CodeBuild\nbuildspec.yml\n - install deps\n - package bundle]
  C -- Artifact (S3 via CodePipeline) --> D[CodeDeploy\nApplication + Deployment Group]
  D --> E[EC2 Instance\ncodedeploy-agent]
  E --> F[Flask app via Gunicorn :8000]

Want a PNG instead of Mermaid?
	•	Open https://mermaid.live, paste the diagram, Export → PNG,
	•	Save to docs/cicd-workflow.png, and reference it in the README:
![CI/CD Workflow](docs/cicd-workflow.png)

⸻

Repository Structure

.
├── src/
│   ├── app.py                 # Flask app (exposes / and /health)
│   └── requirements.txt       # App-specific deps (Flask, Gunicorn)
│
├── scripts/                   # CodeDeploy lifecycle hooks (EC2)
│   ├── before_install.sh      # stop old app, prep dirs
│   ├── after_install.sh       # venv + pip install + systemd unit
│   ├── start.sh               # enable+restart service
│   └── health_check.sh        # curl http://localhost:8000/health
│
├── appspec.yml                # CodeDeploy (EC2) mapping + hooks
├── buildspec.yml              # CodeBuild packaging instructions
├── README.md
└── (optional) docs/cicd-workflow.png


⸻

How the Pipeline Works
	1.	Push to GitHub (tracked branch, e.g., main).
	2.	CodePipeline (Source stage) uses your GitHub OAuth connection to fetch the commit.
	3.	CodeBuild runs buildspec.yml:
	•	installs deps (for tests/packaging),
	•	copies src/, scripts/, and appspec.yml into a bundle,
	•	publishes a single artifact to the CodePipeline S3 artifact store.
	4.	CodeDeploy (Deploy stage) downloads the artifact to the EC2 instance and runs hooks:
	•	BeforeInstall → scripts/before_install.sh
	•	AfterInstall → scripts/after_install.sh
	•	ApplicationStart → scripts/start.sh
	•	ValidateService → scripts/health_check.sh
	5.	The app runs behind systemd at :8000 (gunicorn app:app).

⸻

One‑Time AWS Setup

1) EC2 instance (Amazon Linux 2) + CodeDeploy agent

sudo yum update -y
sudo yum install -y ruby wget
cd /home/ec2-user
wget https://aws-codedeploy-<region>.s3.<region>.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo service codedeploy-agent start
sudo systemctl enable codedeploy-agent

2) IAM roles (minimal to get started)
	•	EC2 instance profile: permissions to talk to CodeDeploy and read artifacts from S3.
(Managed policy often named AWSCodeDeployRoleForEC2 or add equivalent inline permissions + S3 read; add KMS decrypt if your artifact bucket uses a CMK.)
	•	CodePipeline service role: can start CodeBuild/CodeDeploy and pass artifacts.
	•	CodeBuild service role: can read source (provided by pipeline), write artifacts to the pipeline store, and write CloudWatch logs.

3) Developer Tools → Connections (GitHub OAuth)
	•	Create connection → Provider GitHub → authorize via OAuth → grant access to your repo.
	•	Use this connection in CodePipeline Source.

4) CodePipeline (console wizard)
	•	Source: GitHub (select your OAuth connection, repo, branch)
	•	Build: AWS CodeBuild → project uses buildspec.yml
	•	In the CodeBuild project: set Artifacts = CodePipeline
	•	Deploy: AWS CodeDeploy → Application + Deployment Group (targets your EC2)
	•	Deployment config: start with AllAtOnce for single instance.

⸻

Key Config Files (already in this repo)

buildspec.yml

Packages the app so appspec.yml sits at the root of the artifact (required by CodeDeploy):

version: 0.2
phases:
  install:
    commands:
      - pip install -r src/requirements.txt
  build:
    commands:
      - rm -rf bundle
      - mkdir -p bundle
      - cp -r src bundle/src
      - cp appspec.yml bundle/
      - cp -r scripts bundle/scripts
artifacts:
  base-directory: bundle
  files:
    - '**/*'
  discard-paths: no

appspec.yml (EC2)

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

Why this matters: CodeDeploy only looks for appspec.yml at the root of the unzipped artifact.
The buildspec.yml above guarantees that.

⸻

Local Run (optional)

cd src
pip install -r requirements.txt
python app.py
# open http://localhost:8000 (or 5000 if using Flask dev server)


⸻

Troubleshooting Quick Hits
	•	Agent says “Missing credentials”: attach an IAM instance profile to EC2; restart the agent.
	•	“AppSpec file not found”: your artifact is probably double‑zipped. Ensure appspec.yml is at the artifact root (see buildspec.yml above).
	•	requirements.txt not found: this repo uses src/requirements.txt. Either keep it there or update your after_install.sh to check both src/ and app root.
	•	Deploy hangs on ValidateService: increase retries in health_check.sh and check journalctl -u myapp for Gunicorn errors.

⸻

What to Learn Next
	•	Blue/Green with CodeDeploy + ALB
	•	ECS or Lambda deployments (different AppSpec format)
	•	Replace manual EC2 agent install with Launch Template user‑data or a baked AMI
	•	Use OIDC for GitHub Actions → (Hybrid CI in Actions, CD in CodeDeploy)

⸻


Want me to also add a **docs/** folder with a pre‑rendered PNG (I can give you the exact SVG/PNG content or a quick command to export from Mermaid CLI)?
