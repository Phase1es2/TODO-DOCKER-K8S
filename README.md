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
![Mongo image](./screenshot/get_mongo_image.png)
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
![Docker PS Mongo](./screenshot/docker_ps_mongo.png)

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

![Flask App Page](./screenshot/local5000.png)
to verify the application is running locally.

---

## Inspect Docker Volumes

To check your MongoDB volume:

```bash
docker volume inspect mongo_data
docker volume ls
```
![inspect mongo data](./screenshot/inspect_mongo_lc.png)
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
### 4. push to Docker Hub
we can create a new tag and push the image to the Docker Hub
```
docker push majiny/flask-app:latest
```
![Push to DockerHub](./screenshot/push_docker_hub.png)

![Image in DockerHub](./screenshot/docker_hub.png)

## Deploy to the minikube
Since we have minikube downloaded before, so that we can create the deploy to the minikube.

### 1. Start minikube
using the doc to start, minikube service
```
minikube start
```


### 2. create the yaml file
    1. mongodb-pvc.yaml - Persistent Storage for MongoDB:
        - persistentVolumeClaim(PVC)-> A request for storage space in Kubernetes.
        - accessModes: ReadWriteOnce-> The volumen can be mounted as read-write by only one node.
        - resources.requests.storage-> Requests 256 MB of space.,
    MongoDB stores its database files under /data/db. Without a PVC, that data disappears when the container restarts. This ensures data persistence, the database survives redeployments.
    2. mongodb-deployment.yaml - Running MongoDB Pod:
        - kind: Deployment-> Ensures a specific # of identical pods run continuously.
        - metadata.name-> Deployment name (mongodb-deployment).
        - selector/labels-> Used to connect pods, services, and deployments.
        - containers-> Defines which image to run (we used mongo:latest).
        - containerPort-> MongoDB listens on port 27017 (defauly port for MongoDB).
        - aresources.requests-> Reserves 256 MB memory and 0.5 CPU for Mongo.
        - volumeMounts-> Mounts the PVC inside the container at /data/db.
        - volumes-> Attaches the PVC created earlier (mongodb-pvc)
    MongoDB Deployment spins up MongoDB and mounts a persistent volume for storage. Even if the Mongo pod is deleted and recreated, the data remains.
    3. mongodb-service.yaml - Internal Service for MongoDB:
        - kind: Service-> exposes a stable endpoint (DNS name) for Mongo pods.
        - selector.app: mongo-> Connects this Service to pods with label app: mongo.
        - port/ targetPort-> Routes traffic from Service port 27017-> pod port 27017.
    Flask can not directly know a pod's IP (pods are dynamic). Instead, it connects to mongo-service: 27017, a stable hostname managed by Kubernetes DNS.
    4. flask-deployment.yaml - Flask App Deployment:
        - replicas: 1 -> Run on Flask pod (for Minikube demon).
        - images: majiny/flask-app-> The flask app we pushed to Docker Hub before.
        - containerPort: 5000 -> Flask runs internally on port 5000.
        - en variables -> Tell Flask where to find MongoDB.
            - MONGO_HOST=mongo-service
            - MONGO_PORT=27017
        - imagePullyPolicy: IfNotPresent->Uses a cached local image if available
    This defines the applicaiton container. It runs Flask, connects to MongoDB vai service name, and exposes port 5000 inside the cluster.
    5. flask-service.yaml - Exposing Flask to the Outside World:
        - type: LoadBalancer -> Create an external access point/ outside cluster.
        - port: 5000 -> Port exposed to the world.
        - targetPort: 5000 -> Port inside the pod where Flaks runs.
        - selector.app: to-do _> Link to Flaks pod labeld app: to-do
    This lets users access ahte Flask app from outside Kubernetes.

### 3. deploy service
using the code to deploy the service.
```
kubectl apply -f mongodb-pvc.yaml
kubectl apply -f mongodb-service.yaml
kubectl apply -f mongodb-deployment.yaml
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
```
![Deploy On Minikube](./screenshot/mini_dep.png)

### 4. Test the app in Minikube 
To ensure it is running on Minikube by accessing it using the service URL
```
minikube service to-do-service
```
![Minikube service](./screenshot/mini_service.png)

and we can see the URL address is http://127.0.0.1:64879 and we can visited the site by using this URL.
![Minikube App URL](./screenshot/64879_todo.png)

## Deploy to AWS EKS
### 1. download AWS CLI
```
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

$ which aws
/usr/local/bin/aws 
$ aws --version
aws-cli/2.31.9 Python/3.13.7 Darwin/25.1.0 source/arm64
```

### 2. Create cluster
the easy we to create the cluster, we can use
[eksctl][https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html]
```
eksctl create cluster \
  --name flask-mongo-cluster \
  --region us-east-1 \
  --nodes 1 \
  --node-type t3.micro
```

After launch, we can see
![AWS_EKS](./screenshot/AWS_EKS.png)

since there we limited cpu as 500m and memory: 256Mi.
and when we trying to deploy the app, there in no engough resource, so it in the pending state
![Pending State](./screenshot/limited_cpu_memory.png)


using the kubectl to check why it is pending
```
kubectl describe pod <POD_ID>
```
![kubectl_describe](./screenshot/describe_pending.png)


delete the resource we had before and rebuild
```
eksctl delete cluster --name flask-mongo-cluster --region us-east-1


eksctl create cluster \
  --name flask-mongo-cluster \
  --region us-east-1 \
  --nodegroup-name flask-nodegroup \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --node-type t3.small
```
### 3. set up addon
Since we want to use pvc storage
we need to Enable OIDC provbider (required for IAM-> K8s mapping)
```
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster flask-mongo-cluster \
  --approve
```
then create IAM role for the EBS CSI driver
```
eksctl create iamserviceaccount \
  --region us-east-1 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster flask-mongo-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```
Install the AWS EBS CSI driver add-on
```
aws eks create-addon \
  --region us-east-1 \
  --cluster-name flask-mongo-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<id>:role/AmazonEKS_EBS_CSI_DriverRole  #use  the id we set up before 
```
To verify installation
```
kubectl get pods -n kube-system | grep ebs
```
![addon](./screenshot/addon.png)
### 4. deploy to eks
To get pvc running we run same code we had as we deploy to the Minikube
```
kubectl apply -f mongodb-pvc.yaml 
kubectl get pvc
```
![pvc status](./screenshot/pvc_status.png)
we will see there is still no VOLUME, CAPACITY, ACCESS, and MODES
by checking pvc mongo-pvc 
```
kubectl describe pvc mongo-pvc
```
we see the message waiting for first Consumer to be created before binding
![pvc before binding](./screenshot/pvc_b.png)

After we apply mongo-deployment
```
kubectl apply -f mongodbv-deployment.yaml
kubectl get pods
kubectl get pvc
```
![complete PVC](./screenshot/C_PVC.png)

then we apply reset of yaml file
```
kubectl apply -f mongodb-service.yaml
kubectl apply -f flask-deployment.yaml
kubectl apply -f flask-service.yaml
kubectl get all
```
![k8s_aws](./screenshot/kubectl_all_aws.png)

The all nodes are running, and we have a external-ip for our to-do-service
but we can not visited yet, it takes time to set up
！[waiting_external_ip](./screenshot/external_ip.png)

now we can use [http://a0a352132be6044f88cc012031f6faa3-1095562976.us-east-1.elb.amazonaws.com:5000/](http://a0a352132be6044f88cc012031f6faa3-1095562976.us-east-1.elb.amazonaws.com:5000/) to visited our application
![flask_aws](./screenshot/flask_aws.png)


## Deployments and ReplicaSets
Before we change the replicas1
we have defined replicas as 1 
```
spec:
  replicas: 1
```
![get_all_re1](./screenshot/re1_getall.png)

after we change it to 5
```
spec:
  replicas: 5
```
```
kubectl apply -f flask-deployment.yaml
# Verify the ReplicasSet
kubectl get all
kubectl get rs
```
![Verify ReplicasSet](./screenshot/rs_all.png)

When we delete the pod we need to know the pod name
```
kubectl get pods
kubectl delete pod to-do-deployment-7cb67d485c-5vnnk to-do-deployment-7cb67d485c-6c29d
kubectl get pods
```
we will see the two pods were deleted, and by checking pods we will see two new pods created
![auto create](./screenshot/k8s_autocreate.png)


we can scale down the replicas without changing yaml file
```
kubectl scale deployment to-do-deployment --replicas=3
```
![CLI scaled down](./screenshot/scale_down.png)
