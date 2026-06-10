# secure-flask-container-pipeline

A practical beginner lab for taking a small Flask application from local code to a Docker image, publishing it on Docker Hub, and automating the build-scan-push-test workflow with GitHub Actions.

This project focuses on the full delivery path:

```text
Flask app -> Docker image -> Docker Hub -> GitHub Actions -> Trivy scan -> Smoke test
```

---

## Project Snapshot

| Area | What This Project Covers |
|---|---|
| Application | Python Flask web app with HTML page and JSON health endpoint |
| Containerization | Dockerfile, `.dockerignore`, local build and run |
| Registry | Docker Hub login, image tagging, push, pull, and sharing |
| Automation | GitHub Actions workflow for CI/CD |
| Security | Non-root container user, build context hygiene, Trivy vulnerability scan |
| Validation | Browser checks, health endpoint, smoke tests in CI |

---

## Repository Layout

```text
.
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── templates/
│   └── index.html
└── .github/
    └── workflows/
        └── docker-build-push.yml
```

---

## What the App Does

The Flask app exposes two endpoints:

| Endpoint | Purpose |
|---|---|
| `/` | Displays a simple welcome page |
| `/health` | Returns JSON showing app health, node name, and timestamp |

Example health response:

```json
{
  "node": "developer",
  "status": "healthy",
  "time": "2026-06-09T10:30:00Z"
}
```

The app reads `APP_NODE` from the environment, so the displayed node name can change without editing the source code.

---

## Requirements

Install or create accounts for:

- Git
- Docker
- GitHub account
- Docker Hub account
- Code editor such as VS Code

Check local tools:

```bash
git --version
docker --version
```

Confirm Docker works:

```bash
docker run hello-world
```

---

## Step 1: Try a Public Image First

Before building the Flask app image, the lab starts by running a public Nginx container.

Pull the image:

```bash
docker pull nginx:alpine
```

Run it:

```bash
docker run -d -p 8080:80 --name my-nginx nginx:alpine
```

Check the container:

```bash
docker ps
docker logs my-nginx
```

Open in the browser:

```text
http://localhost:8080
```

Clean up:

```bash
docker rm -f my-nginx
```

This step proves the pull-run-test lifecycle before building a custom image.

---

## Step 2: Build the Flask Image Locally

Build the image:

```bash
docker build -t my-web-app .
```

Run the container:

```bash
docker run -d -p 5000:5000 --name webapp-test my-web-app
```

Test in the browser:

```text
http://localhost:5000
http://localhost:5000/health
```

Stop and remove the test container:

```bash
docker rm -f webapp-test
```

---

## Dockerfile Highlights

The Dockerfile is designed with beginner-friendly security practices:

```dockerfile
FROM python:3.11-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
```

Important choices:

- `python:3.11-slim` keeps the base image smaller than the full Python image.
- `COPY requirements.txt .` happens before copying the app code to improve Docker layer caching.
- `--no-cache-dir` prevents pip from storing package cache inside the image.
- `USER appuser` avoids running the application as root.

---

## Build Context Protection

The `.dockerignore` file keeps unwanted files out of the image build context.

```dockerignore
__pycache__
*.pyc
*.pyo
.git
.env
*.md
Dockerfile
.dockerignore
```

This helps prevent local-only files, Git history, and secrets from being copied into the Docker image.

---

## Step 3: Push the Image to Docker Hub

Create a Docker Hub access token first:

```text
Docker Hub -> Account Settings -> Security -> New Access Token
```

Use the token when logging in:

```bash
docker login -u your-dockerhub-username
```

Tag the image:

```bash
docker tag my-web-app your-dockerhub-username/my-web-app:v1.0.0-yourname
```

Push it:

```bash
docker push your-dockerhub-username/my-web-app:v1.0.0-yourname
```

Recommended tag format:

```text
version-owner
```

Example:

```text
v1.0.0-john
```

---

## Step 4: Pull and Run the Published Image

Remove the local tagged image if you want to test as a fresh user:

```bash
docker rmi your-dockerhub-username/my-web-app:v1.0.0-yourname
```

Pull from Docker Hub:

```bash
docker pull your-dockerhub-username/my-web-app:v1.0.0-yourname
```

Run it:

```bash
docker run -d -p 5000:5000 --name pulled-app your-dockerhub-username/my-web-app:v1.0.0-yourname
```

Verify:

```bash
curl http://localhost:5000
curl http://localhost:5000/health
```

---

## Step 5: Configure GitHub Secrets

The GitHub Actions workflow needs Docker Hub credentials.

Add these repository secrets:

| Secret Name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Your Docker Hub access token |

Location:

```text
GitHub repo -> Settings -> Secrets and variables -> Actions
```

Do not commit Docker Hub tokens directly into the repository.

---

## Step 6: Automated DevSecOps Workflow

The workflow lives at:

```text
.github/workflows/docker-build-push.yml
```

It runs on:

- Pushes to `main`
- Pull requests targeting `main`

Pipeline flow:

```text
Checkout code
    |
Docker Hub login
    |
Build image locally
    |
Scan with Trivy
    |
Push image if scan passes
    |
Pull published image
    |
Run smoke tests
    |
Clean up container
```

---

## CI/CD Security Gate

The workflow uses Trivy to scan the image before it is pushed.

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@0.20.0
  with:
    image-ref: "${{ env.DOCKER_HUB_USER }}/${{ env.IMAGE_NAME }}:v1.0.0-${{ env.TAG_SUFFIX }}"
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    severity: 'HIGH,CRITICAL'
```

The important part:

```yaml
exit-code: '1'
```

If Trivy finds matching vulnerabilities, the workflow fails and the image is not pushed.

---

## CI Smoke Tests

After the image is pushed, the workflow pulls it back and tests it.

```bash
curl -f http://localhost:5000/health
curl -s http://localhost:5000/ | grep -i "hello"
```

These checks confirm:

- The container starts successfully
- The `/health` endpoint responds
- The home page returns expected content

---

## How to Trigger the Pipeline

Commit and push changes:

```bash
git add .
git commit -m "Add Flask Docker DevSecOps lab"
git push
```

Then open:

```text
GitHub repo -> Actions
```

You should see the workflow running.

---

## Verification Checklist

Use this checklist to confirm the lab is complete.

| Check | Command or Location |
|---|---|
| Nginx public image runs | `http://localhost:8080` |
| Flask image builds locally | `docker build -t my-web-app .` |
| Flask container runs locally | `http://localhost:5000` |
| Health endpoint works | `http://localhost:5000/health` |
| Docker Hub login works | `docker login -u username` |
| Image pushed successfully | Docker Hub repository page |
| Image can be pulled back | `docker pull username/my-web-app:tag` |
| GitHub secrets added | Repository Actions secrets |
| Workflow starts on push | GitHub Actions tab |
| Trivy scan runs | Workflow logs |
| Smoke tests pass | Workflow logs |

---

## Cleanup

Remove containers:

```bash
docker rm -f $(docker ps -aq) 2>/dev/null
```

Remove lab images:

```bash
docker rmi my-web-app your-dockerhub-username/my-web-app:v1.0.0-yourname nginx:alpine 2>/dev/null
```

---

## Skills Practiced

- Pulling images from Docker Hub
- Running containers with port mappings
- Writing a Flask app
- Creating a Dockerfile
- Using `.dockerignore`
- Building and tagging Docker images
- Pushing images to Docker Hub
- Pulling published images
- Creating GitHub repository secrets
- Writing a GitHub Actions workflow
- Running container vulnerability scans
- Smoke testing a published image

---

## Possible Next Improvements

- Add unit tests before image build
- Add image tags based on Git commit SHA
- Generate an SBOM
- Sign images with Cosign
- Add Dependabot for Python dependencies
- Publish to GitHub Container Registry
- Deploy the image to Kubernetes
- Add branch protection rules
- Require passing CI before merging pull requests

---

## License

This project is for learning and practice. Use, fork, and improve it freely.
