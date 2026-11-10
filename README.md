\# Flask + MongoDB on Kubernetes



A containerized Flask application connected to an authenticated MongoDB database, deployed on Kubernetes using Minikube. This project demonstrates persistent storage, Kubernetes service discovery, resource management, and CPU-based horizontal autoscaling.



\## Architecture



\- **\*\*Flask:\*\*** REST API running on port \`5000\`.

\- **\*\*MongoDB:\*\*** Authenticated database running as a StatefulSet.

\- **\*\*Docker:\*\*** Containerizes the Flask application.

\- **\*\*Kubernetes:\*\*** Manages application deployment, networking, and scaling.

\- **\*\*PersistentVolume (PV) and PersistentVolumeClaim (PVC):\*\*** Provide persistent MongoDB storage.

\- **\*\*HorizontalPodAutoscaler (HPA):\*\*** Automatically scales Flask replicas based on CPU utilization.

\- **\*\*Kubernetes Secret:\*\*** Stores MongoDB credentials and the connection URI.



\## Project structure



\`\`\`text

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

\`\`\`



The MongoDB Secret is created separately and intentionally excluded from version control.



\## Prerequisites



\- Python 3.9+

\- Docker Desktop

\- kubectl

\- Minikube

\- Git

\- Postman or another API testing tool

\- Docker Hub account (only if you want to publish the image)



\## 1. Clone the repository



\`\`\`powershell

git clone https\://github.com/SubhadeepAtta/flask-mongodb-app.git

cd flask-mongodb-app

\`\`\`



\## 2. Run the application locally (optional)



Create and activate a virtual environment:



\`\`\`powershell

python -m venv venv

.\venv\Scripts\Activate.ps1

\`\`\`



Install dependencies:



\`\`\`powershell

pip install -r requirements.txt

\`\`\`



Ensure MongoDB is running locally, then start Flask:



\`\`\`powershell

python app.py

\`\`\`



\## 3. Build the Docker image



From the project root:



\`\`\`powershell

docker build -t flask-mongodb-app:v1 .

\`\`\`



Verify that the image was created:



\`\`\`powershell

docker images

\`\`\`



\### Load the image into Minikube



The Kubernetes Flask Deployment uses the local image \`flask-mongodb-app:v1\` with \`imagePullPolicy: IfNotPresent\`.



Minikube has its own image environment, so building an image in Docker Desktop does not automatically make it available inside Minikube.



After starting Minikube, load the image:



\`\`\`powershell

minikube image load flask-mongodb-app:v1

\`\`\`



This step is required when following the local-image deployment configuration in \`k8s/flask.yaml\`.



\### Push the image to Docker Hub (optional)



Replace \`YOUR_DOCKERHUB_USERNAME\` with your Docker Hub username.



\`\`\`powershell

docker login

docker tag flask-mongodb-app:v1 YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1

docker push YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1

\`\`\`



If you want Kubernetes to pull the published image instead of using the locally loaded image, update the \`image\` field in \`k8s/flask.yaml\`:



\`\`\`yaml

image: YOUR_DOCKERHUB_USERNAME/flask-mongodb-app:v1

imagePullPolicy: IfNotPresent

\`\`\`



\## 4. Start Kubernetes using Minikube



Start Docker Desktop first.



Start the Minikube cluster:



\`\`\`powershell

minikube start --driver=docker

\`\`\`



Enable the Metrics Server, which is required by the HPA to obtain CPU utilization metrics:



\`\`\`powershell

minikube addons enable metrics-server

\`\`\`



Load the Flask image into Minikube:



\`\`\`powershell

minikube image load flask-mongodb-app:v1

\`\`\`



Verify the cluster:



\`\`\`powershell

kubectl get nodes

kubectl top nodes

\`\`\`



The Metrics Server may take a short time to become ready. If metrics are initially unavailable, wait and retry \`kubectl top nodes\`.



\## 5. Configure MongoDB credentials



Create the Kubernetes Secret locally. Replace the example password with your own password.



\`\`\`powershell

kubectl create secret generic mongodb-secret --from-literal=MONGO_INITDB_ROOT_USERNAME=admin --from-literal=MONGO_INITDB_ROOT_PASSWORD=ChangeMe123 --from-literal=MONGODB_URI="mongodb://admin:ChangeMe123\@mongodb:27017/flask_db?authSource=admin"

\`\`\`



The username and password in the MongoDB URI must match the credentials provided to the MongoDB container.



The \`authSource=admin\` parameter tells MongoDB to authenticate the root user against the \`admin\` database.



**\*\*Important:\*\*** Do not commit credentials or the actual Secret configuration to GitHub. The Secret is intentionally excluded through \`.gitignore\`.



\## 6. Deploy Kubernetes resources



Apply the resources in the following order:



\`\`\`powershell

kubectl apply -f k8s/mongodb-pv.yaml

kubectl apply -f k8s/mongodb-pvc.yaml

kubectl apply -f k8s/mongodb.yaml

kubectl apply -f k8s/flask.yaml

kubectl apply -f k8s/hpa.yaml

\`\`\`



Verify the deployment:



\`\`\`powershell

kubectl get pods

kubectl get services

kubectl get pv,pvc

kubectl get hpa

\`\`\`



Wait until the MongoDB and Flask pods are in the \`Running\` state before testing the application.



\## 7. Access the Flask application



The Flask Service is configured as a NodePort Service. You can also access it locally using port forwarding:



\`\`\`powershell

kubectl port-forward service/flask-service 5000:5000

\`\`\`



Open:



\`http\://localhost:5000/\`



\### API endpoints



\| Method | Endpoint | Description |

\|---|---|---|

\| GET | \`/\` | Returns a welcome message and current time |

\| POST | \`/data\` | Inserts JSON data into MongoDB |

\| GET | \`/data\` | Retrieves stored records from MongoDB |



Example POST request to \`http\://localhost:5000/data\`:



\`\`\`json

{

  "name": "Kubernetes Test",

  "message": "MongoDB integration successful",

  "version": 1

}

\`\`\`



Expected response:



\`\`\`json

{

  "status": "Data inserted"

}

\`\`\`



Send a GET request to \`/data\` to retrieve the stored records.



\## 8. Kubernetes DNS and service discovery



Kubernetes provides an internal DNS system that allows services to communicate using their names instead of fixed IP addresses.



The Flask application connects to MongoDB using the Service hostname:



\`\`\`text

mongodb:27017

\`\`\`



The full MongoDB Service DNS name is:



\`\`\`text

mongodb.default.svc.cluster.local

\`\`\`



The DNS name consists of:



\- \`mongodb\`: MongoDB Service name.

\- \`default\`: Kubernetes namespace.

\- \`svc\`: Identifies a Kubernetes Service.

\- \`cluster.local\`: Default Kubernetes cluster DNS domain.



When Flask connects to \`mongodb:27017\`, Kubernetes DNS resolves the Service name to the MongoDB Service IP. The Service then forwards traffic to the MongoDB pod.



The MongoDB Service is configured as \`ClusterIP\`, which makes it accessible internally within the cluster rather than directly from outside.



The Flask Service selects Flask pods using the \`app: flask-app\` label and distributes incoming traffic across the available replicas.



\## 9. Resource requests and limits



Both Flask and MongoDB containers use the following resource configuration:



\| Resource | Request | Limit |

\|---|---:|---:|

\| CPU | 200m | 500m |

\| Memory | 250M | 500M |



**\*\*Requests\*\*** define the resources Kubernetes uses for scheduling decisions and resource allocation.



**\*\*Limits\*\*** define the maximum CPU and memory resources a container can consume.



These settings help prevent containers from consuming excessive cluster resources and provide predictable resource allocation.



\## 10. Horizontal Pod Autoscaler (HPA)



The Flask Deployment uses an HPA configured with:



\| Setting | Value |

\|---|---:|

\| CPU utilization target | 70% |

\| Minimum replicas | 2 |

\| Maximum replicas | 5 |



The HPA monitors average CPU utilization relative to the CPU requests configured for the Flask containers.



Check autoscaling:



\`\`\`powershell

kubectl get hpa

kubectl top pods

\`\`\`



When CPU utilization remains above the configured target, Kubernetes can increase the number of Flask replicas. When the load decreases, the HPA can scale the replicas down, subject to its scaling behavior and stabilization settings.



\## 11. Testing and results



The following scenarios were tested in Minikube:



\| Test scenario | Result |

\|---|---|

\| Flask welcome endpoint | Passed |

\| MongoDB POST operation | Passed |

\| MongoDB GET operation | Passed |

\| Authenticated Flask-to-MongoDB connection | Passed |

\| PV/PVC binding | Passed |

\| MongoDB pod recreation and data persistence | Passed |

\| HPA scale-up under load | Passed |

\| HPA scale-down after stopping traffic | Passed |



\### Autoscaling test



During load testing:



\- Initial Flask replicas: 2.

\- CPU utilization increased to approximately 110% against the 70% target.

\- HPA increased the replica count from 2 to 4.

\- After stopping the load generators, CPU utilization dropped.

\- The replica count eventually returned to 2.



This demonstrated that the HPA could scale the Flask Deployment in response to increased CPU demand and scale it down after the load stopped.



\### MongoDB persistence test



A test record was inserted into MongoDB. The MongoDB pod was then deleted and recreated by its StatefulSet.



The previously inserted records were still available after the pod restarted.



This verified that MongoDB data was stored on the persistent volume rather than only in the container filesystem.



\### Issues encountered and resolutions



**\*\*1. Docker image unavailable inside Minikube\*\***



Initially, the MongoDB image was available in Docker Desktop but was not available in Minikube's image environment.



Resolution:



\`\`\`powershell

minikube image load mongo:latest

\`\`\`



The image pull policy was configured as \`IfNotPresent\` so Kubernetes could use the locally loaded image.



The Flask image was also loaded into Minikube using:



\`\`\`powershell

minikube image load flask-mongodb-app:v1

\`\`\`



**\*\*2. Metrics unavailable for HPA\*\***



After enabling the Metrics Server, CPU metrics were not immediately available.



Resolution:



Waited for the Metrics Server to become ready and verified metric collection using:



\`\`\`powershell

kubectl top nodes

kubectl top pods

\`\`\`



**\*\*3. HPA did not scale during initial low-load testing\*\***



The initial test traffic did not generate enough CPU utilization to cross the configured 70% target.



Resolution:



Additional load-generator pods were used to increase incoming traffic. CPU utilization subsequently reached approximately 110%, and the HPA scaled the Flask Deployment from 2 to 4 replicas.



**\*\*4. HPA scale-down was not immediate\*\***



After stopping the load generators, the replica count did not immediately return to its minimum.



Resolution:



The HPA was allowed time to observe the reduced CPU utilization and complete its scale-down behavior. The replica count eventually returned to 2.



\### Testing Evidence

The following screenshots demonstrate the API functionality and Kubernetes autoscaling behavior.

**1. POST `/data` — Successful data insertion**

![POST API response](screenshots/post-data.png)

The API returned `201 Created`, confirming successful insertion of the JSON record into MongoDB.

**2. GET `/data` — Successful data retrieval**

![GET API response](screenshots/get-data.png)

The API returned `200 OK` and retrieved the previously inserted record.

**3. HPA — Autoscaling under load**

![HPA autoscaling output](screenshots/hpa-autoscaling.png)

During load testing, CPU utilization reached 110% against the configured 70% target. The HPA increased the Flask replica count from 2 to 4. After the load decreased, CPU utilization dropped to 69%.

The replica count eventually returned to 2 after the load generators were stopped.

## 12. Design choices and alternatives



\| Component | Chosen approach | Alternative considered | Reason for choosing the implemented approach |

\|---|---|---|---|

\| Flask workload | Deployment | StatefulSet | Flask instances are stateless and do not require stable pod identities or persistent storage. |

\| MongoDB workload | StatefulSet | Deployment | MongoDB benefits from stable pod identity and persistent storage association. |

\| MongoDB Service | ClusterIP | NodePort / LoadBalancer | The database should only be accessible from within the Kubernetes cluster. |

\| Flask Service | NodePort | ClusterIP only | NodePort provides an external access mechanism for the Flask application. Port forwarding was used for local testing. |

\| Persistent storage | PV and PVC using hostPath | Dynamic volume provisioning | hostPath is sufficient for this single-node Minikube environment and demonstrates persistent storage configuration. |

\| Scaling | HPA based on CPU utilization | Manual replica scaling | HPA automatically adjusts the number of Flask replicas based on CPU demand. |

\| Credentials | Kubernetes Secret | Hardcoded credentials in application code | Secrets separate credentials from application code and Kubernetes manifests committed to the repository. |



\### Possible production improvements



The current implementation is intended for a local Minikube environment. A production deployment could be improved by using:



\- Managed MongoDB or a production-grade database deployment.

\- A dedicated secret-management solution.

\- TLS for secure communication.

\- Readiness and liveness probes.

\- A production WSGI server instead of Flask's development server.

\- Dynamic persistent volume provisioning instead of hostPath storage.

\- A container image registry for reproducible deployments.



\## 13. Cleanup



To remove the deployed Kubernetes resources:



\`\`\`powershell

kubectl delete -f k8s/hpa.yaml

kubectl delete -f k8s/flask.yaml

kubectl delete -f k8s/mongodb.yaml

kubectl delete -f k8s/mongodb-pvc.yaml

kubectl delete -f k8s/mongodb-pv.yaml

kubectl delete secret mongodb-secret

\`\`\`



**\*\*Note:\*\*** The PersistentVolume uses a \`Retain\` reclaim policy. Review and manually clean up its stored data if you intend to remove it permanently.