# CI/CD Pipeline Lab

In this lab you will build a working CI/CD pipeline using two containers running on your machine:
a **Gitea** server (self-hosted Git) and a **Jenkins** server. Every time you push code, Jenkins
will automatically build a Docker image and push it to your Docker Hub account.

```
[Your machine]  -->  git push  -->  [Gitea]
                                       |
                                   webhook
                                       |
                                       v
                                  [Jenkins]
                                       |
                              docker build & push
                                       |
                                       v
                                 [Docker Hub]
```

---

## Prerequisites

Install all of the following before starting:

| Tool | Download |
|---|---|
| Docker Desktop for Windows | https://www.docker.com/products/docker-desktop |
| Git for Windows | https://git-scm.com/download/win |
| A Docker Hub account | https://hub.docker.com (free) |

Open **Docker Desktop** and make sure it is running (the whale icon in the taskbar should be
still, not animated). Keep it open for the entire lab.

> **WSL2 note:** Docker Desktop on Windows uses the WSL2 backend. You do not need to install
> WSL2 manually — Docker Desktop will set it up for you if needed.

---

## Part 1 — Start the infrastructure

Open **PowerShell** (not CMD) and navigate to the `cicd-lab` folder you received:

```powershell
cd cicd-lab
docker compose up --build -d
```

The first run takes a few minutes because Docker is downloading the base images and installing
Jenkins plugins. When it finishes, check that both containers are running:

```powershell
docker compose ps
```

You should see `gitea` and `jenkins` with status `running`.

Open your browser and confirm both UIs load:
- Gitea → http://localhost:3000
- Jenkins → http://localhost:8080

---

## Part 2 — Set up Gitea

### 2.1 — First-time configuration

Open http://localhost:3000. Gitea will show a one-time setup page.

Change only these two fields, leave everything else as default:

| Field | Value |
|---|---|
| Site Title | CI/CD Lab |
| Administrator Username | `gitadmin` |
| Administrator Password | (choose one you will remember) |
| Administrator Email | (any email) |

Click **Install Gitea**.

### 2.2 — Create a repository

1. Click the **+** button (top right) → **New Repository**
2. Fill in:
   - **Repository name:** `my-app`
   - **Visibility:** Public
3. Click **Create Repository**

You will land on the empty repo page. Copy the HTTP clone URL shown — it looks like:
```
http://localhost:3000/gitadmin/my-app.git
```

### 2.3 — Push the student app

In PowerShell, go into the `student-app` folder and initialize a Git repo:

```powershell
cd student-app
git init
git add .
git commit -m "initial commit"
git remote add origin http://localhost:3000/gitadmin/my-app.git
git push -u origin main
```

Git will ask for your Gitea username and password. Refresh the Gitea page — you should see
your files listed.

---

## Part 3 — Set up Jenkins

### 3.1 — Unlock Jenkins

Open http://localhost:8080. Jenkins will ask for an initial admin password. Retrieve it with:

```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the output and paste it into the browser.

### 3.2 — Initial setup wizard

1. Click **Install suggested plugins** and wait for the installation to finish.
2. Create your first admin user (username, password, email).
3. On the "Instance Configuration" page, leave the URL as `http://localhost:8080/` and click
   **Save and Finish**.

### 3.3 — Add your Docker Hub credentials

Jenkins needs your Docker Hub username and password to push images.

1. Go to **Dashboard → Manage Jenkins → Credentials**
2. Click **(global)** under "Stores scoped to Jenkins"
3. Click **Add Credentials** (left sidebar)
4. Fill in:
   | Field | Value |
   |---|---|
   | Kind | Username with password |
   | Username | your Docker Hub username |
   | Password | your Docker Hub password |
   | ID | `dockerhub-credentials` |
   | Description | Docker Hub |
5. Click **Create**

> **Important:** the ID must be exactly `dockerhub-credentials` — that is what the Jenkinsfile
> references.

### 3.4 — Create the pipeline job

1. Go to **Dashboard → New Item**
2. Enter name: `my-app-pipeline`
3. Select **Pipeline** and click **OK**
4. On the configuration page, scroll down to **Build Triggers**
   - Check **Build when a change is pushed to Gitea**
5. Scroll down to **Pipeline** and set:
   | Field | Value |
   |---|---|
   | Definition | Pipeline script from SCM |
   | SCM | Git |
   | Repository URL | `http://gitea:3000/gitadmin/my-app.git` |
   | Branch | `*/main` |
   | Script Path | `Jenkinsfile` |

   > Note: the URL uses `gitea` (the container name), not `localhost`. Jenkins is inside Docker
   > and reaches Gitea over the internal network.

6. Click **Save**

### 3.5 — Run the pipeline manually (first time)

Click **Build Now** in the left sidebar. Watch the build in **Stage View** — it should go through
three green stages: **Checkout → Build → Push**.

If it fails, click the build number → **Console Output** to read the error.

After a successful run, go to https://hub.docker.com and confirm your image appears there.

---

## Part 4 — Configure the webhook

Now make it automatic: a push to Gitea should trigger Jenkins without any manual step.

1. In Gitea, open your `my-app` repository
2. Go to **Settings → Webhooks → Add Webhook → Gitea**
3. Fill in:
   | Field | Value |
   |---|---|
   | Target URL | `http://jenkins:8080/gitea-webhook/post` |
   | HTTP Method | POST |
   | Content Type | application/json |
   | Trigger on | Push events |
4. Click **Add Webhook**
5. On the webhook list page, click **Test Delivery** — you should see a green `200` response.

> The URL uses `jenkins` (the container name) because the webhook request travels inside the
> Docker network from Gitea to Jenkins.

---

## Part 5 — Push a change and watch the pipeline

Edit `student-app/app.py` and change the message in the `hello()` function. Then push:

```powershell
git add app.py
git commit -m "update greeting"
git push
```

Switch to Jenkins (http://localhost:8080) — within a few seconds a new build should appear
automatically in your pipeline. Watch it run through all three stages.

Check Docker Hub again: you will see a new image tag (the build number) alongside `latest`.

---

## Cleanup

To stop and remove the containers (data volumes are preserved):

```powershell
docker compose down
```

To also delete all stored data (Gitea repos, Jenkins config) and start fresh:

```powershell
docker compose down -v
```

---

## Troubleshooting

**Jenkins cannot connect to Gitea when cloning the repo**
Make sure the Repository URL in the Jenkins job uses `http://gitea:3000/...` and not
`http://localhost:3000/...`. Jenkins runs inside Docker and cannot reach `localhost`.

**Webhook test returns a non-200 status**
Check that the Gitea plugin is installed in Jenkins: go to **Manage Jenkins → Plugins →
Installed** and search for "Gitea". If missing, install it and restart Jenkins.

**`docker: command not found` in Jenkins build output**
The Jenkins image did not build correctly. Run `docker compose down` then
`docker compose up --build -d` to rebuild.

**`permission denied` accessing `/var/run/docker.sock`**
In Docker Desktop, go to **Settings → General** and make sure
**"Expose daemon on tcp://localhost:2375 without TLS"** is NOT checked (that is for a different
use case). The WSL2 socket should be available automatically. Restart Docker Desktop and rerun
`docker compose up -d`.

**Jenkins asks for a password when pushing to Gitea**
The pipeline does not push to Gitea — only you do from your machine. Jenkins only reads
from Gitea (clones the repo). Make sure the Gitea repo is set to **Public**.
