# springboot-cicd-demo

Minimal Spring Boot project for learning CI/CD.

## What this project includes

- `GET /hello` returns `hello`
- `GET /actuator/health` returns application health
- Maven wrapper, so local Maven install is not required
- Dockerfile for container packaging
- GitHub Actions CI for test and package
- GitHub Actions CD for publishing a Docker image to GHCR
- GitHub Actions deployment workflow for a single Ubuntu VM

## Project requirements

- JDK 17
- Git
- Docker Desktop or Docker Engine
- GitHub account

## Local development

### 1. Run tests

```bash
./mvnw test
```

### 2. Build the jar

```bash
./mvnw package
```

Generated artifact:

```bash
target/springboot-cicd-demo-0.0.1-SNAPSHOT.jar
```

### 3. Run the application locally

```bash
java -jar target/springboot-cicd-demo-0.0.1-SNAPSHOT.jar
```

### 4. Verify the endpoints

```bash
curl http://127.0.0.1:8080/hello
curl http://127.0.0.1:8080/actuator/health
```

Expected results:

- `/hello` returns `hello`
- `/actuator/health` returns `{"status":"UP"}`

## Docker

### 1. Make sure Docker is running

Check both the Docker client and daemon:

```bash
docker version
docker info
```

If you see an error like:

```bash
failed to connect to the docker API at unix:///var/run/docker.sock
```

start Docker Desktop or Docker Engine first, then retry.

### 2. Build the jar

The Dockerfile copies the packaged jar from `target/`, so package the application first:

```bash
./mvnw package -DskipTests
```

### 3. Build the image

```bash
docker build -t springboot-cicd-demo:local .
```

### 4. Run the container

```bash
docker run --rm -p 8080:8080 springboot-cicd-demo:local
```

### 5. Verify inside Docker

```bash
curl http://127.0.0.1:8080/hello
curl http://127.0.0.1:8080/actuator/health
```

Expected results:

- `/hello` returns `hello`
- `/actuator/health` returns `{"status":"UP"}`

### 6. Stop the container

Press `Ctrl+C` in the terminal running `docker run`.

## GitHub setup

### 1. Create a GitHub repository

Example:

```bash
git init
git add .
git commit -m "init springboot cicd demo"
git branch -M main
git remote add origin <your-repo-url>
git push -u origin main
```

### 2. CI workflow

File:

```bash
.github/workflows/ci.yml
```

What CI does:

- runs on `push` and `pull_request`
- sets up JDK 17
- caches Maven dependencies
- runs `./mvnw test`
- runs `./mvnw package -DskipTests`
- uploads the jar as a workflow artifact

### 3. CD workflow

File:

```bash
.github/workflows/cd.yml
```

What CD does:

- runs automatically after `CI` succeeds on `main`
- can also be triggered manually from GitHub Actions
- builds the application jar
- builds a Docker image
- pushes the image to GitHub Container Registry (`ghcr.io`)

CD prerequisites:

- the repository default branch should be `main`
- GitHub Actions must be enabled for the repository
- repository package permissions must allow publishing to GHCR
- the workflow uses the built-in `GITHUB_TOKEN`, so no extra GHCR password is required for the push step

Published image format:

```bash
ghcr.io/<github-username-or-org>/springboot-cicd-demo:main
ghcr.io/<github-username-or-org>/springboot-cicd-demo:<commit-sha>
```

To pull the published image locally:

```bash
docker pull ghcr.io/<github-username-or-org>/springboot-cicd-demo:main
docker run --rm -p 8080:8080 ghcr.io/<github-username-or-org>/springboot-cicd-demo:main
```

### 4. VM deployment workflow

File:

```bash
.github/workflows/deploy-vm.yml
```

What deployment does:

- runs automatically after the `CD` workflow succeeds on `main`
- can also be triggered manually from GitHub Actions with `workflow_dispatch`
- connects to your Ubuntu VM over SSH
- logs in to `ghcr.io` on the VM
- pulls `ghcr.io/<github-username-or-org>/springboot-cicd-demo:main`
- removes the old container if it exists
- starts the new container on port `8080`
- checks `http://127.0.0.1:8080/actuator/health` on the VM

Required repository secrets:

- `VM_HOST`: your VPS public IP or domain
- `VM_USER`: SSH username, for example `deployer`
- `VM_SSH_KEY`: private key used by GitHub Actions to SSH into the VPS
- `VM_PORT`: optional, use `22` if you do not use a custom SSH port
- `GHCR_USERNAME`: GitHub username used to read GHCR packages
- `GHCR_TOKEN`: GitHub personal access token with at least `read:packages`

## Hostinger Ubuntu VPS setup

These commands are for a minimal Docker-based deployment target on Ubuntu.

### 1. Install Docker

```bash
sudo apt update
sudo apt install -y docker.io curl
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

### 2. Create a deployment user

```bash
sudo adduser deployer
sudo usermod -aG docker deployer
sudo mkdir -p /opt/springboot-cicd-demo
sudo chown -R deployer:deployer /opt/springboot-cicd-demo
```

After adding the user to the `docker` group, log out and log back in before testing Docker commands.

### 3. Add the GitHub Actions SSH public key

On your local machine:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy"
```

Then append the generated public key to the VPS:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "<paste-public-key-here>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Store the matching private key in the GitHub repository secret `VM_SSH_KEY`.

### 4. Open firewall ports

Make sure your Hostinger firewall or Ubuntu firewall allows:

- `22/tcp` for SSH
- `8080/tcp` for the application

If you use `ufw`, for example:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
```

### 5. Verify the VPS manually

Before relying on GitHub Actions, verify the target machine can run Docker:

```bash
docker ps
curl http://127.0.0.1:8080/actuator/health
```

The `curl` command will only succeed after the first deployment.

## GitHub Actions verification

After pushing to GitHub:

1. Open the repository `Actions` tab
2. Confirm `CI` passed
3. Confirm `CD` passed after `CI`
4. Confirm `Deploy To VM` passed after `CD`
5. Open the workflow artifact from `CI`
6. Open the package list and verify the image was pushed to `ghcr.io`
7. If needed, open the package settings and change visibility from private to public
8. Open `http://<your-vps-ip>:8080/hello`
9. Open `http://<your-vps-ip>:8080/actuator/health`

## Current local verification status

Completed:

- `./mvnw test`
- `./mvnw package -DskipTests`
- local `java -jar` verification for `/hello` and `/actuator/health`

Current blocker for Docker verification:

- Docker CLI is installed, but the local daemon is not reachable at `unix:///var/run/docker.sock`
- until Docker Desktop or Docker Engine is started, `docker build` and `docker run` cannot be completed locally

## Current learning scope

This version intentionally keeps things minimal:

- no database
- no Kubernetes
- no Tekton
- no Argo CD
- single VM deployment only

The next step is either:

- harden the single VM deployment with rollback, HTTPS, and a reverse proxy
- or migrate the CI/CD flow toward Tekton and Argo CD
