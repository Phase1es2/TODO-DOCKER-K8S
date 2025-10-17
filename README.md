 # 🐳 Docker + MongoDB + Flask Setup Guide

This guide explains how to prepare, install, and test a Flask application connected to MongoDB using Docker.

---

## Preparation

### 1. Download the Source Code
Download the project source code from the provided Google Drive folder.

---

### 2. Install Docker (macOS)

Follow [Docker Desktop installation for Mac](https://docs.docker.com/desktop/setup/install/mac-install/).

Alternatively, using command line:

```bash
sudo hdiutil attach Docker.dmg
sudo /Volumes/Docker/Docker.app/Contents/MacOS/install
sudo hdiutil detach /Volumes/Docker
```

Check Docker version:

```bash
docker --version
```

---

### 3. Install Kubernetes Tools (Optional)

Install **kubectl** and **minikube** using Homebrew:

```bash
brew install kubectl
brew install minikube
```

Verify installations:

```bash
kubectl --help
minikube --help
```

---

## Test on Local Machine

### 1. Set Up the Environment
Unzip the downloaded file, navigate into the project folder, and install dependencies:

```bash
pip install -r requirements.txt
```

---

### 2. Run MongoDB Using Docker

Pull the official MongoDB image:

```bash
docker pull mongo
```

Run the MongoDB container:

```bash
docker run -d   --name mongodb   -p 27017:27017   -v mongo_data:/data/db   mongo
```

**Explanation:**
- `-d` → Run in detached mode (background)
- `--name mongodb` → Container name
- `-p 27017:27017` → Expose MongoDB default port
- `-v mongo_data:/data/db` → Persist data using Docker volume
- `mongo` → Use the official MongoDB image

Check if the container is running:

```bash
docker ps
```

---

### 3. MongoDB Configuration in `app.py`

The Flask app connects to MongoDB as follows:

```python
mongodb_host = os.environ.get('MONGO_HOST', 'localhost')
mongodb_port = int(os.environ.get('MONGO_PORT', '27017'))
client = MongoClient(mongodb_host, mongodb_port)
db = client.camp2016
todos = db.todo
```

---

### 4. Start the Flask Application

Run the Flask app:

```bash
python3 app.py
```

Then open your browser and go to:

[http://127.0.0.1:5000](http://127.0.0.1:5000)

to verify the application is running locally.

---

## Inspect Docker Volumes

To check your MongoDB volume:

```bash
docker volume inspect mongo_data
docker volume ls
```

---

## Inspect MongoDB Data

Access MongoDB shell inside the container:

```bash
docker exec -it mongodb mongosh
```

Then run:

```bash
use camp2016                # switch to database
show collections            # list collections
db.todo.find().pretty()     # view data inside 'todo' collection
```

---

**You’ve successfully set up Docker, MongoDB, and Flask locally!**


## Containize 

### 1.Create a Dockfile for Flask App
```
# Use the official Python 3.8 slim image as the base image
FROM python:3.8-slim

# Set the working directory within the container
WORKDIR /app

COPY requirements.txt .

# Upgrade pip and install Python dependencies
RUN pip3 install --upgrade pip && pip install --no-cache-dir -r requirements.txt

# Copy the rest of app
COPY . .

# Expose Flask port default
EXPOSE 5000

CMD ["python", "app.py"]
```

### 2.build the image
```
docker build -t flask-app .
```

### 3. buil the conatiner
```
docker run -d -p 5000:5000 flask-app flask-app
```

### bug
 -  MacOS using port 5000 for airplay, need to shut down it to use 5000 port
 - pymongo.errors.ServerSelectionTimeoutError: localhost:27017: [Errno 111] Connection refused
    - inside the flask container, localhost means the Flask containner itself, not the host or Mongo cantainer.
    ```
    client = MongoClient('localhost', 27017)
    ```

### fix it
    - using docker-compose.yml
    - for localtest manual docker run with --network
```
#create a bridge network name flasknet, so conatiners on the same network can communicate by name
docker network create flasknet
docker run -d --name mongodb --network flasknet -p 27017:27017 mongo
docker run -d --name flask-app --network flasknet -p 5000:5000 flasknet -e MONGO_HOST=mongodb flask-app
```
