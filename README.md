# Flask + MongoDB on Kubernetes

A containerized Flask application connected to an authenticated MongoDB database, deployed on Kubernetes using Minikube. The project demonstrates persistent storage, service discovery, resource management, and CPU-based horizontal autoscaling.

## Architecture

- **Flask:** REST API running on port 5000.
- **MongoDB:** Authenticated database running as a StatefulSet.
- **Docker:** Containerizes the Flask application.
- **Kubernetes:** Manages application deployment and networking.
- **PersistentVolume (PV) and PersistentVolumeClaim (PVC):** Provide persistent MongoDB storage.
- **HorizontalPodAutoscaler (HPA):** Automatically scales Flask replicas based on CPU utilization.

## Project structure

```text
flask-mongodb-app/
├── app.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── README.md
└── k8s/
    ├── mongodb-pv.yaml
    ├── mongodb-pvc.yaml
    ├── mongodb.yaml
    ├── flask.yaml
    └── hpa.yaml
```

The MongoDB Secret is created separately and is intentionally excluded from version control.

## Prerequisites

- Python 3.9+
- Docker Desktop
- kubectl
- Minikube
- Docker Hub account (for publishing the image)
- Postman or another API testing tool

## 1. Run the application locally

Clone the repository:

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd flask-mongodb-app
```

Create and activate a virtual environment:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
pip install -r requirements.txt
```

Run Flask:

```powershell
python app.py
```

## 2. Build the Docker image

From the project root:

```powershell
docker build -t flask-mongodb-app:v1 .
```

Verify:

```powershell
docker images
```

### Push the image to Docker Hub

Replace `YOUR_DOCKERHUB_USERNAME` with your Docker Hub username.

```powershell
docker login
docker tag flask-mongodb-app:v1 YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1
docker push YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1
```

Update the `image` field in `k8s/flask.yaml` to:

```yaml
image: YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1
imagePullPolicy: IfNotPresent
```

## 3. Start Kubernetes using Minikube

Start Docker Desktop, then run:

```powershell
minikube start --driver=docker
```

Enable the Metrics Server:

```powershell
minikube addons enable metrics-server
```

Verify the cluster:

```powershell
kubectl get nodes
```

## 4. Configure MongoDB credentials

Create the Kubernetes Secret locally. Replace the example password with your own password.

```powershell
kubectl create secret generic mongodb-secret --from-literal=MONGO_INITDB_ROOT_USERNAME=admin --from-literal=MONGO_INITDB_ROOT_PASSWORD=ChangeMe123 --from-literal=MONGODB_URI="mongodb://admin:ChangeMe123@mongodb:27017/flask_db?authSource=admin"
```

Use the same username and password in the MongoDB URI.

Do not commit credentials or the actual Secret configuration to GitHub.

## 5. Deploy Kubernetes resources

Apply the resources in this order:

```powershell
kubectl apply -f k8s/mongodb-pv.yaml
kubectl apply -f k8s/mongodb-pvc.yaml
kubectl apply -f k8s/mongodb.yaml
kubectl apply -f k8s/flask.yaml
kubectl apply -f k8s/hpa.yaml
```

Verify deployment:

```powershell
kubectl get pods
kubectl get services
kubectl get pv,pvc
kubectl get hpa
```

## 6. Access the Flask application

Forward the Flask Service port:

```powershell
kubectl port-forward service/flask-service 5000:5000
```

Open:

`http://localhost:5000/`

### API endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` | Returns welcome message and current time |
| POST | `/data` | Inserts JSON data into MongoDB |
| GET | `/data` | Retrieves stored records |

Example POST request:

```json
{
  "name": "Kubernetes Test",
  "message": "MongoDB integration successful",
  "version": 1
}
```

Expected response:

```json
{
  "status": "Data inserted"
}
```

## 7. Kubernetes DNS and service discovery

Kubernetes provides an internal DNS system that allows services to communicate using their names rather than fixed IP addresses.

The Flask application connects to MongoDB using:

`mongodb:27017`

The full MongoDB Service DNS name is:

`mongodb.default.svc.cluster.local`

Here:
- `mongodb` is the Service name.
- `default` is the Kubernetes namespace.
- `svc` identifies a Kubernetes Service.
- `cluster.local` is the default cluster DNS domain.

The MongoDB Service is configured as ClusterIP, making it accessible internally within the cluster rather than directly from outside.

The Flask Service selects Flask pods using the `app: flask-app` label.

## 8. Resource requests and limits

Both Flask and MongoDB containers use the following configuration:

| Resource | Request | Limit |
|---|---:|---:|
| CPU | 200m | 500m |
| Memory | 250Mi | 500Mi |

Requests define the minimum resources Kubernetes uses for scheduling decisions. Limits define the maximum resources a container can consume.

These settings help prevent containers from consuming excessive cluster resources.

## 9. Horizontal Pod Autoscaler

The Flask Deployment uses an HPA configured with:

| Setting | Value |
|---|---:|
| CPU utilization target | 70% |
| Minimum replicas | 2 |
| Maximum replicas | 5 |

Check autoscaling:

```powershell
kubectl get hpa
kubectl top pods
```

The HPA increases or decreases Flask replicas based on average CPU utilization relative to the requested CPU resources.

## 10. Testing and results

The following scenarios were tested in Minikube:

| Test | Result |
|---|---|
| Flask welcome endpoint | Passed |
| MongoDB POST operation | Passed |
| MongoDB GET operation | Passed |
| Flask-to-MongoDB authenticated connection | Passed |
| PV/PVC binding | Passed |
| MongoDB pod restart and data persistence | Passed |
| HPA scale-up under load | Passed |
| HPA scale-down after stopping traffic | Passed |

### Autoscaling test

During load testing:
- Initial Flask replicas: 2
- CPU utilization reached 110% against a 70% target.
- HPA increased the replica count to 4.
- After stopping the load generators, CPU utilization dropped and replicas returned to 2.

### Persistence test

A test record was inserted into MongoDB. The MongoDB pod was deleted and recreated by its StatefulSet. The previously inserted records were still available afterward.

This verified that MongoDB data was stored on the persistent volume rather than only in the container filesystem.

## 11. Design choices

- **StatefulSet for MongoDB:** Provides stable pod identity and persistent storage association.
- **Deployment for Flask:** Supports replicas and horizontal scaling.
- **ClusterIP for MongoDB:** Keeps the database internal to the Kubernetes cluster.
- **NodePort for Flask:** Provides an external access mechanism.
- **PV/PVC:** Separates persistent storage from the MongoDB container lifecycle.
- **HPA:** Automatically adjusts Flask capacity based on CPU demand.
- **Kubernetes Secret:** Keeps database credentials separate from application configuration.

For production, additional improvements could include managed MongoDB, stronger secret management, TLS, readiness and liveness probes, and a production WSGI server.

## 12. Cleanup

To remove the deployed Kubernetes resources:

```powershell
kubectl delete -f k8s/hpa.yaml
kubectl delete -f k8s/flask.yaml
kubectl delete -f k8s/mongodb.yaml
kubectl delete -f k8s/mongodb-pvc.yaml
kubectl delete -f k8s/mongodb-pv.yaml
kubectl delete secret mongodb-secret
```

**Note:** The PV has a `Retain` reclaim policy. Review and manually clean up its stored data if you intend to remove it permanently.
