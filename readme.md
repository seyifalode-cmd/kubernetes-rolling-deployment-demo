**Kubernetes Rolling Deployment Demo** — deploying a Spring Boot application to Kubernetes and demonstrating zero-downtime rolling updates across versioned container images.

---

## Project at a Glance

| | |
|---|---|
| **Tools Used** | Kubernetes (kubectl), Docker, Maven, Spring Boot, DockerHub |
| **Platform** | Kubernetes cluster (GKE or EC2-hosted); DockerHub container registry |
| **Languages** | Java (Spring Boot), YAML, Dockerfile |
| **What It Does** | Builds a Spring Boot REST API, packages it as a Docker image, pushes it to DockerHub, deploys it to Kubernetes, and demonstrates a rolling update strategy that replaces pods incrementally without dropping traffic |

---

## The Problem This Project Solves

Deploying updated application code to production is one of the most operationally risky moments in a software system's lifecycle. A naive approach — stop all old instances, then start all new ones — creates downtime. Even a few seconds of unavailability is unacceptable for production services with real traffic.

Rolling updates solve this by replacing pods incrementally rather than all at once. Kubernetes removes a percentage of old pods, starts new ones, waits for them to pass readiness checks, and then continues the process until all replicas are running the new version. Throughout the update, a mix of old and new pods is serving traffic, and the load balancer routes requests only to healthy instances. If the new version fails its readiness checks, the rollout pauses automatically, protecting production.

This project demonstrates the full lifecycle: building the application with Maven, containerizing it with Docker, pushing the image to DockerHub, deploying it to Kubernetes using a declarative deployment manifest, and verifying that the rolling update strategy is configured and functional. The deployment manifest specifies `maxSurge: 25%` and `maxUnavailable: 25%`, which means at most one extra pod is created above the desired count and at most one pod is taken down at any time during an update — keeping the service live throughout.

---

## Architecture

```
Developer Machine
      |
      | mvn clean install
      v
+------------------+
|  Maven Build     |
|  target/*.jar    |
+------------------+
      |
      | docker build
      v
+------------------+
|  Docker Image    |
|  spring-boot-app |
+------------------+
      |
      | docker push
      v
+------------------+
|  DockerHub       |
|  sirjobzy/       |
|  spring-boot-app |
+------------------+
      |
      | kubectl apply
      v
+-------------------------------------------+
|  Kubernetes Cluster                       |
|                                           |
|  Deployment: springboot-deployment        |
|  Replicas: 4                              |
|  Strategy: RollingUpdate                  |
|    maxSurge: 25%  maxUnavailable: 25%     |
|                                           |
|  [Pod v2] [Pod v2] [Pod v2] [Pod v2]      |
|                                           |
|  Service: springboot-app-service          |
|  Type: LoadBalancer  Port: 80 -> 8080     |
+-------------------------------------------+
      |
      | External LoadBalancer IP
      v
Browser / Client
```

---

## Repository Structure

```
kubernetes-rolling-deployment-demo/
├── README.md
├── Dockerfile                      # Multi-stage build: IBM JRE 8 base, copies Maven jar
├── pom.xml                         # Maven project descriptor
├── spring-boot-deployment.yml      # Kubernetes Deployment + LoadBalancer Service manifest
└── src/
    └── main/
        ├── java/com/example/springboot/
        │   ├── Application.java    # Spring Boot entry point
        │   └── HelloController.java # REST endpoints: /, /DevOps (GET + POST), /YorkU (POST)
        └── resources/
            └── application.properties
```

---

## Walkthrough

### Step 1: Build the application with Maven

```bash
mvn clean install
```

This compiles the Java source, runs tests, and produces a JAR under `target/`. The Dockerfile copies this JAR into the container image.

### Step 2: Build and tag the Docker image

```bash
docker build -t sirjobzy/spring-boot-app:2 .
```

The Dockerfile uses IBM JRE 8 as the base image and exposes port 8080:

```dockerfile
FROM ibmjava:8-jre
COPY target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "/app.jar"]
```

### Step 3: Push the image to DockerHub

```bash
docker login
docker push sirjobzy/spring-boot-app:2
```

### Step 4: Deploy to Kubernetes

```bash
kubectl apply -f spring-boot-deployment.yml
```

The manifest creates a Deployment with 4 replicas and a LoadBalancer Service. The rolling update strategy is declared explicitly:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
  type: RollingUpdate
```

### Step 5: Verify the deployment

```bash
kubectl get pods
kubectl get replicaset
kubectl get deployments
kubectl get services
```

Note the external IP address assigned to `springboot-app-service`. Wait until `EXTERNAL-IP` is no longer `<pending>`.

### Step 6: Verify the application is live

```bash
curl http://<EXTERNAL-IP>/
# Returns: Greetings from Spring Boot - vers 2!!

curl http://<EXTERNAL-IP>/DevOps
# Returns: Flow; fast feedback; learning & experimentation
```

### Step 7: Trigger a rolling update

To update to a new image version, update the `image` field in the deployment manifest and re-apply:

```bash
# Edit spring-boot-deployment.yml: change image tag from :2 to :3
kubectl apply -f spring-boot-deployment.yml

# Watch the rolling update proceed
kubectl rollout status deployment/springboot-deployment
```

---

## How to Reproduce

Prerequisites: Maven, Docker, a DockerHub account, and a Kubernetes cluster with `kubectl` configured.

```bash
# Clone the repository
git clone <repo-url>
cd kubernetes-rolling-deployment-demo

# Build the artifact
mvn clean install

# Build and push the Docker image (substitute your DockerHub username)
docker build -t <your-username>/spring-boot-app:2 .
docker login
docker push <your-username>/spring-boot-app:2

# Update the image reference in spring-boot-deployment.yml to match your DockerHub path
# Then deploy
kubectl apply -f spring-boot-deployment.yml

# Monitor rollout
kubectl rollout status deployment/springboot-deployment
kubectl get pods
kubectl get services

# Test the endpoint
curl http://<EXTERNAL-IP>/
curl http://<EXTERNAL-IP>/DevOps

# Clean up
kubectl delete -f spring-boot-deployment.yml
```

---

## Key Observations

- The `minReadySeconds: 60` field in the deployment spec requires each new pod to remain healthy for 60 seconds before Kubernetes considers it available. This slows down the rollout intentionally to allow monitoring systems time to detect failures before they propagate across all replicas.
- The `annotations.kubernetes.io/change-cause` field records the reason for each rollout. This value appears in `kubectl rollout history` and is essential for audit trails in regulated environments.
- Rolling updates are reversible: `kubectl rollout undo deployment/springboot-deployment` reverts to the previous ReplicaSet, restoring the old image version without rebuilding anything.

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
