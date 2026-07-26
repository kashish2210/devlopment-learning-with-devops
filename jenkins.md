# Jenkins Learning Notes

## 1. Setting Up Jenkins with Docker

### Base Setup
Jenkins with Blue Ocean + Docker support is typically built as a custom image on top of the official `jenkins/jenkins` image, adding Docker CLI and plugins.

**Key lesson: Plugin version vs Jenkins core version compatibility**

Plugins installed via `jenkins-plugin-cli` pull the *latest* versions of their dependencies. If your base Jenkins image is old (e.g. `2.414.2`), but plugin dependencies require a much newer core (e.g. `2.504.1+`), the build fails with errors like:

```
blueocean-config (1.27.25) requires a greater version of Jenkins (2.479.3) than 2.414.2
```

**Fix:** Use a current Jenkins LTS base image.

```dockerfile
FROM jenkins/jenkins:2.516.1-jdk17
```

> Dockerfile instructions (`FROM`, `RUN`, etc.) go **inside the Dockerfile**, not typed directly into the terminal. Typing `FROM ...` into bash gives `command not found`.

### Example Dockerfile Pattern
```dockerfile
FROM jenkins/jenkins:2.516.1-jdk17
USER root
RUN apt-get update && apt-get install -y lsb-release python3 python3-pip
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
    https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
    https://download.docker.com/linux/debian bookworm stable" \
    > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.3 docker-workflow:1.28"
```

### Build & Run Commands
```bash
docker build -t myjenkins-blueocean:2.516.1-1 .

docker network create jenkins

docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.516.1-1
```

- `jenkins-data` volume → persists jobs, config, plugins across container recreation.
- Port `8080` → web UI. Port `50000` → JNLP agent connections.

### Common Docker Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `network jenkins not found` | Network never created | `docker network create jenkins` |
| `network with name jenkins already exists` | Already created | Just proceed, ignore |
| `Conflict. The container name ... already in use` | Stale container from failed run | `docker rm <name>` then retry |
| `failed to copy: httpReadSeeker ... EOF` | Transient CDN/CloudFront download failure | Retry pull; use a loop (see below) |

**Retry loop for flaky pulls:**
```bash
for i in 1 2 3 4 5; do
  docker pull <image> && break
  echo "attempt $i failed, retrying..."
  sleep 10
done
```

---

## 2. Managing the Jenkins Container

```bash
docker ps               # check running containers
docker stop jenkins-blueocean
docker start jenkins-blueocean
docker restart jenkins-blueocean
docker logs -f jenkins-blueocean
```

Stopping does **not** trigger `--restart=on-failure` — that only applies to crashes, not manual stops. Data persists via the named volume regardless of stop/start/remove (as long as volume isn't deleted).

### Getting Initial Admin Password (first-time setup)
```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

### Resetting Admin Password (if locked out)
```bash
docker exec -it -u root jenkins-blueocean bash
cat > /var/jenkins_home/init.groovy.d/reset-admin.groovy << 'EOF'
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.getInstance()
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount("admin", "newpassword123")
instance.setSecurityRealm(hudsonRealm)
instance.save()
EOF
exit
docker restart jenkins-blueocean
```
Delete the groovy script afterward so it doesn't reset the password on every restart.

---

## 3. Executing Commands Inside the Container

```bash
docker exec -it jenkins-blueocean bash
```
- `-i` = interactive (keep STDIN open)
- `-t` = allocate a TTY (needed for a usable shell prompt)
- Without `-t`, the shell won't behave properly.

**Default user is `jenkins`, not root** — no `sudo` available. To install packages, exec in as root explicitly:
```bash
docker exec -it -u root jenkins-blueocean bash
apt-get update && apt-get install -y python3 python3-pip
```

> Anything installed this way is **not persistent** — it lives only in the container's writable layer and is lost if the container is recreated. Bake it into the Dockerfile for permanence.

---

## 4. Jenkins Plugin Management

### Missing Plugin Dependency Example
```
Token Macro Plugin (477...) 
Plugin is missing: json-path-api (2.9.0-148...)
```
One missing dependency cascades into many "failed to load" errors for everything depending on it.

**Fix options:**
1. Search **Manage Jenkins → Plugins → Available plugins** for the exact plugin name.
2. If not found, use **Advanced settings → Deploy Plugin** to upload a `.hpi` file downloaded directly:
   ```
   https://updates.jenkins.io/download/plugins/<plugin-name>/<version>/<plugin-name>.hpi
   ```
3. Or bake it directly into the Dockerfile's plugin-cli install line — most reliable long-term:
   ```dockerfile
   RUN jenkins-plugin-cli --plugins "blueocean:1.25.3 docker-workflow:1.28 json-path-api"
   ```

### Installing Plugins via Jenkins CLI

Download CLI jar:
```bash
curl -O http://localhost:8080/jnlpJars/jenkins-cli.jar
```

Requires Java on the **host machine** (not inside the container):
```bash
sudo apt update
sudo apt install openjdk-17-jre-headless
java -version
```

Install a plugin:
```bash
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth <user>:<API_TOKEN> install-plugin docker-workflow -restart
```

- Use an **API Token** (generated via user icon → Configure → API Token), not your raw password.
- `Handshake error` / WebSocket errors during CLI use are usually caused by bad/missing auth credentials, not actual connection problems.
- **Never share tokens in chat/plaintext** — if exposed, revoke and regenerate immediately.

For a one-off plugin install, the web UI (**Manage Jenkins → Plugins → Available plugins**) is simpler than the CLI.

---

## 5. Running Jobs & Common Build Errors

### Workspaces Are Isolated Per Job
Each Jenkins job gets its own folder:
```
/var/jenkins_home/workspace/<job-name>/
```
Nothing is shared between jobs automatically. If a job has no SCM (source control) configured, its workspace stays empty — files from another job's workspace are **not** visible to it.

### Shell Scripting Basics
```bash
# WRONG - treats BUILD_ID as a command to execute
BUILD_ID
echo the build id is $BUILD_ID

# RIGHT
echo "the build id is $BUILD_ID"
```
A bare variable name (without `$`) is interpreted as a command, not a value reference.

### File Not Found Errors
Usually caused by:
- Typos in filenames (case-sensitive on Linux — `hellowworld.py` ≠ `helloworld.py`)
- Running from the wrong directory (check with `pwd`, `ls`)
- Job has no SCM checkout configured, so the file was never pulled into the workspace

---

## 6. Installing Software Inside Jenkins Containers

To install Python inside a running container (temporary, non-persistent):
```bash
docker exec -it -u root jenkins-blueocean bash
apt-get update && apt-get install -y python3 python3-pip
```

For permanence, add to Dockerfile:
```dockerfile
RUN apt-get update && apt-get install -y python3 python3-pip
```

**Key mental model:** Jenkins is not a single monolithic environment. The **controller** (`jenkins-blueocean` container) and any **build agents** (spun up separately, e.g. via Docker Cloud) are completely separate containers with separate filesystems and separately installed software. Installing Python on the controller does **not** make it available to agent containers.

---

## 7. Proxy Servers

### Forward Proxy
Sits in front of **clients**. Client → Proxy → Server. The server doesn't know who the real client is.

**Example used in this setup:** `tecnativa/docker-socket-proxy` — sits between Jenkins (client) and the real Docker daemon socket, restricting which API operations are allowed via environment variables, instead of giving Jenkins unrestricted root-level Docker access.

```bash
docker run -d \
  --name docker-socket-proxy \
  --restart=always \
  --network jenkins \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -e CONTAINERS=1 \
  -e IMAGES=1 \
  -e NETWORKS=1 \
  -e SERVICES=1 \
  -e VOLUMES=1 \
  -e POST=1 \
  -e VERSION=1 \
  -e INFO=1 \
  -e PING=1 \
  -e EXEC=1 \
  -e ALLOW_START=1 \
  -e ALLOW_STOP=1 \
  -e ALLOW_RESTARTS=1 \
  tecnativa/docker-socket-proxy
```

**No need to publish this to the host** (`-p`) if Jenkins only needs to reach it over the internal Docker network — connect via `tcp://docker-socket-proxy:2375` using the container name.

**Key env vars and why they matter:**
| Var | Purpose |
|---|---|
| `CONTAINERS=1` | Allow read (list/inspect) container operations |
| `POST=1` | Allow write operations (create/start containers) |
| `EXEC=1` | Allow exec into containers (needed for some connect methods) |
| `ALLOW_START/STOP/RESTARTS=1` | Allow container lifecycle actions |
| `VERSION=1`, `INFO=1`, `PING=1` | Allow basic Docker API health/version checks |

Without `POST`/`EXEC`/`ALLOW_START` etc., Jenkins gets `403 Forbidden` when trying to actually launch agent containers, even if the initial `Test Connection` succeeds (since that only needs `VERSION`).

### Reverse Proxy
Sits in front of **servers**. Client → Reverse Proxy → Backend Server(s). The client doesn't know which real backend handled the request.

Used for: load balancing, SSL/TLS termination, hiding internal architecture, caching, routing by URL path. Examples: Nginx, HAProxy, Traefik, Cloudflare.

**Quick distinction:** Forward proxy protects/represents the *client*. Reverse proxy protects/represents the *server*.

---

## 8. Setting Up Jenkins Docker Cloud (Dynamic Build Agents)

### Purpose
Instead of running builds on the Jenkins controller itself, Docker Cloud lets Jenkins spin up **temporary, isolated containers** as build agents on demand, then destroy them after the build.

### Configuration Steps (Manage Jenkins → Nodes → Clouds → Add a new cloud → Docker)

**Docker Host URI:**
```
tcp://docker-socket-proxy:2375
```
(points at the socket proxy container over the `jenkins` network)

**Server credentials:** `- none -` (proxy has no TLS/auth, protected by network isolation instead)

**Enabled:** must be checked, or the cloud is inactive.

**Container Cap:** limits how many agent containers can run simultaneously.

### Docker Agent Template
- **Labels:** e.g. `docker-agent` — jobs target this label via `agent { label 'docker-agent' }`
- **Docker Image:** e.g. `jenkins/inbound-agent:alpine-jdk21`
- **Remote File System Root:** e.g. `/home/jenkins/agent`
- **Connect method:** **JNLP** is the standard choice for `jenkins/inbound-agent` images (container proactively connects back to Jenkins).
- **Pull strategy:** "Pull once and update latest" (avoids re-pulling every build) or "Never pull" once cached locally.
- **Network** (under Container settings): **must be set to `jenkins`**, or the spawned agent container won't be attached to the same Docker network as the Jenkins controller, and DNS resolution to the controller will fail.

### Critical Networking Gotcha
Agent containers spun up by Docker Cloud are **not automatically attached** to the same custom Docker network as the controller unless explicitly configured. Symptom:

```
java.net.UnknownHostException: jenkins-blueocean
INFO: Could not locate server among [http://jenkins-blueocean:8080/]
```

**Fix:** set the **Network** field in the Docker Agent template's Container settings to the network name (`jenkins`). This is what actually enables container-name-based DNS resolution between the agent and the controller.

### Jenkins URL Setting
Check **Manage Jenkins → System → Jenkins Location → Jenkins URL**. If set to `http://localhost:8080/`, agent containers won't be able to resolve it (localhost inside a container refers to itself). Should match the controller's container name on the shared network, e.g. `http://jenkins-blueocean:8080/`.

---

## 9. Custom Agent Images

Default `jenkins/inbound-agent` images are minimal (Java + agent only) — no Python, no other tooling. Since agent containers are stateless/ephemeral, anything installed manually via `docker exec` disappears on the next build.

**Fix: build a custom agent image with the tools baked in.**

```dockerfile
FROM jenkins/inbound-agent:alpine-jdk21
USER root
RUN apk add --no-cache python3 py3-pip
USER jenkins
```

```bash
docker build -t jenkins-agent-python:alpine-jdk21 .
```

Then in the Docker Agent template, set:
- **Docker Image:** `jenkins-agent-python:alpine-jdk21`
- **Pull strategy:** "Never pull" (since it's a local-only image, not on a registry)

---

## 10. Stateless vs Stateful (Concept)

| | Stateless | Stateful |
|---|---|---|
| Remembers past requests? | No | Yes |
| Data location | Passed in each request / external DB | Stored locally in the instance |
| Scaling | Easy — clone freely | Harder — needs shared/synced storage |
| Example in this setup | Docker build agents (destroyed after each build) | Jenkins controller (persists via named volume) |

Docker Cloud build agents are stateless by design — nothing installed inside them persists, which is exactly why tooling must be baked into the image itself rather than installed ad hoc.

---

## 11. Writing Jenkinsfiles (Declarative Pipelines)

Example pipeline used in this setup:

```groovy
pipeline {
    agent { 
        node {
            label 'docker-agent-python'
        }
    }
    triggers {
        pollSCM '*/5 * * * *'
    }
    stages {
        stage('Build') {
            steps {
                echo "Building.."
                sh '''
                cd myapp
                pip install --break-system-packages -r requirements.txt
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Testing.."
                sh '''
                cd myapp
                python3 hello.py
                python3 hello.py --name=Brad
                '''
            }
        }
        stage('Deliver') {
            steps {
                echo 'Deliver....'
                sh '''
                echo "doing delivery stuff.."
                '''
            }
        }
    }
}
```

### Cron Syntax for `pollSCM` / `triggers`
5 whitespace-separated fields:
```
MINUTE HOUR DAY_OF_MONTH MONTH DAY_OF_WEEK
```
Every 5 minutes:
```
*/5 * * * *
```
`H/5 * * * *` (using `H` instead of `*`) is Jenkins' recommended pattern for spreading load across jobs, rather than triggering all polling jobs at the exact same instant.

**Common error:** `*/5****` (no spaces) → "You appear to be missing whitespace between * and *." Each field must be separated by a space.

### `externally-managed-environment` pip error
Newer Python (PEP 668) blocks system-wide `pip install` by default to avoid conflicts with the OS package manager (`apk`/`apt`).

```
error: externally-managed-environment
```

**Fix (quick, safe for ephemeral containers):**
```bash
pip install --break-system-packages -r requirements.txt
```

**Fix (cleaner, longer-term):** use a virtual environment — but then every subsequent `sh` step needs to re-activate it, since each step runs in a fresh shell:
```bash
python3 -m venv venv
. venv/bin/activate
pip install -r requirements.txt
```
```bash
# in later stages:
. venv/bin/activate
python3 hello.py
```

### `ModuleNotFoundError: No module named 'pipes'`
Caused by using an old package version incompatible with newer Python. The `pipes` module was removed in Python 3.13+. Example: `fire==0.4.0` internally imports `pipes`.

**Fix:** bump the package version in `requirements.txt`:
```
fire>=0.5.0
```
Older pinned dependencies can silently break when the Python version in the agent image is updated — always check changelogs when bumping base images.

---

## 12. Quick Reference: Full Working Setup Recap

1. Build Jenkins controller image (`jenkins/jenkins:2.516.1-jdk17` base + Docker CLI + plugins).
2. Run it with a named volume for persistence, on a custom Docker network (`jenkins`).
3. Run `docker-socket-proxy` on the same network, with the right permission env vars (`CONTAINERS`, `POST`, `EXEC`, `ALLOW_START/STOP/RESTARTS`, etc.).
4. Configure a Docker Cloud in Jenkins pointing at `tcp://docker-socket-proxy:2375`.
5. Add a Docker Agent template with the correct label, image, JNLP connect method, and — critically — the **Network** field set to `jenkins`.
6. Set the correct **Jenkins URL** (`http://jenkins-blueocean:8080/`) so agents can find the controller.
7. Build a custom agent image with any needed tooling (e.g. Python) baked in, since agents are stateless/ephemeral.
8. Write a Jenkinsfile targeting the custom agent's label, handling `pip`'s externally-managed-environment restriction and correct package versions.

This is the complete path from a broken `docker-plugin-cli` build to a working, containerized, dynamically-provisioned Jenkins CI pipeline.