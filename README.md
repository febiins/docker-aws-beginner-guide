# docker-aws-beginner-guide

# Dockerizing and Deploying a Flask App to AWS EC2 (Beginner Guide)

A step-by-step walkthrough of containerizing a simple Flask app with Docker and deploying it to AWS EC2 — written for anyone doing this for the first time.

## Why this repo exists

I'm a full-stack developer learning cloud infrastructure and DevOps practices. This documents my first hands-on Docker + AWS deployment, broken down simply enough for someone with zero prior Docker/AWS experience to follow along and actually understand *why* each step exists, not just copy-paste it.

## The app

A minimal Flask API with two routes — a homepage and a simple in-memory task list. Small on purpose, so the focus stays on the Docker/AWS process, not the app itself.

```python
# app.py
from flask import Flask, jsonify, request

app = Flask(__name__)
tasks = []

@app.route('/')
def home():
    return jsonify({"message": "Task Tracker API is running!"})

@app.route('/tasks', methods=['GET'])
def get_tasks():
    return jsonify(tasks)

@app.route('/tasks', methods=['POST'])
def add_task():
    task = request.json.get('task')
    tasks.append(task)
    return jsonify({"added": task, "all_tasks": tasks}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```
# requirements.txt
Flask==3.0.3
```

---

## Part 1: Containerizing it with Docker

### The Dockerfile

```dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

**What each line does, in plain terms:**

| Line | What it does |
|---|---|
| `FROM python:3.11-slim` | Starts from an official lightweight image that already has Python installed |
| `ENV PYTHONDONTWRITEBYTECODE=1` | Skips creating `.pyc` cache files — pointless in a container that gets rebuilt from scratch each time |
| `ENV PYTHONUNBUFFERED=1` | Makes logs show up immediately instead of being buffered — important for debugging with `docker logs` |
| `WORKDIR /app` | Sets `/app` as the "current folder" inside the container for every command that follows |
| `COPY requirements.txt .` then `RUN pip install` | Installs dependencies *before* copying the rest of the code — this is a caching trick, so code changes don't force a full dependency reinstall every rebuild |
| `COPY . .` | Copies the rest of the project files in |
| `EXPOSE 5000` | Documents which port the app listens on (informational — the actual port mapping happens at `docker run`) |
| `CMD ["python", "app.py"]` | The command that runs when the container starts |

### `.dockerignore`

Excludes files that shouldn't be copied into the image:

```
__pycache__/
*.pyc
venv/
env/
.git/
.gitignore
*.log
```

### Build and test it locally

```bash
docker build -t demo-flask-app .
docker run -p 5000:5000 demo-flask-app
```

Then visit `http://localhost:5000` — you should see `{"message": "Task Tracker API is running!"}`.

**Common beginner mistake:** the app must bind to `host='0.0.0.0'`, not `127.0.0.1` or `localhost`, inside `app.run()`. Binding to `127.0.0.1` inside a container makes it unreachable from outside the container — a very common first-time Docker networking trip-up.

---

## Part 2: Pushing the image to Docker Hub

```bash
docker login
docker tag demo-flask-app <your-dockerhub-username>/demo-flask-app
docker push <your-dockerhub-username>/demo-flask-app
```

`docker tag` just renames your local image to Docker Hub's required `username/imagename` format — it doesn't rebuild anything. `docker push` is the actual upload.

---

## Part 3: Deploying to AWS EC2

### 1. Launch an EC2 instance

- AMI: **Ubuntu Server 22.04/24.04 LTS** (free-tier eligible)
- Instance type: **t3.micro** (free-tier eligible)
- Create a new key pair for SSH access when prompted

### 2. Open port 5000 in the security group

By default, AWS blocks all incoming traffic except SSH (port 22). Think of the security group as a security guard at every door — each port is a separate locked door, and opening one (SSH) doesn't open any others.

- Go to the instance's **Security** tab → click the security group link
- **Inbound rules** → **Edit inbound rules** → **Add rule**
- Type: `Custom TCP`, Port range: `5000`, Source: `Anywhere-IPv4 (0.0.0.0/0)`
- Save

### 3. Connect to the instance

Easiest method for beginners: on the instance page, click **Connect** → **EC2 Instance Connect** tab → **Connect**. This opens a terminal directly in your browser — no `.pem` file setup needed.

### 4. Install Docker on the instance

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

The last line lets you run `docker` commands without typing `sudo` every time — but it only takes effect on a **new** login session, so close the terminal tab and reconnect after running it.

Verify with:
```bash
docker ps
```
(Should show an empty table, no permission errors.)

### 5. Pull and run the image

```bash
docker pull <your-dockerhub-username>/demo-flask-app
docker run -d -p 5000:5000 --name task-tracker <your-dockerhub-username>/demo-flask-app
```

- `-d` runs it in the background
- `-p 5000:5000` maps the instance's port 5000 to the container's port 5000
- `--name task-tracker` gives it a memorable name instead of a random one

### 6. See it live

Find the instance's **Public IPv4 address** on its details page, then visit:

```
http://<your-ec2-public-ip>:5000
```

You should see the same JSON response as your local test — except now it's reachable from anywhere on the internet, not just your own machine.

---

## Key concepts recap

- **Image vs container:** an image is a static blueprint; a container is a running instance of it. One image, many containers.
- **Security groups** are AWS's firewall — every port is closed by default and must be opened deliberately.
- **`-aG` in `usermod -aG docker ubuntu`**: the `-a` (append) matters — without it, you'd overwrite all the user's existing group memberships instead of adding to them.
- Your EC2 instance's public IP **changes** every time you stop and restart it, unless you set up an Elastic IP.

## Stack used

Docker, Docker Hub, AWS EC2, Ubuntu, Python, Flask
