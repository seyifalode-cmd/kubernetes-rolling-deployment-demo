# Spring Boot App to demo rolling deployment feature in K8s.

This readme and code relates to branch version2.  There are two branches, which represent two different scenarios.
**version2** is the initial build and deployment of the application code
**version3** is an update to the deployment definition to support Rolling Upgrades, in addition to a small change to the application code to create different output for this version3 where a subsequent set of intructions can be found.

## Create an EC2 where the code can be built and then pushed to DockerHub
This terraform-ansible code also installs docker.
https://github.com/YorkU-SCS-Cloud-Ops/Module6_Class3_Repo1_EC2ForJava.git

Clone this repo [Module6_Class8_Repo1_AppForRollingUpdate] and cd into the cloned folder to the level of the pom file.

Build the code.
```bash
mvn clean install
```

Create a docker image using the built artifact.  The Dockerfile in this repo has all the required details.
```bash
docker build -t shanemannion/spring-boot-app:vers-2
```

## Deploy the built image to Kubernetes.
Copy the contents of spring-boot-deployment.yml to create a file on the Google Cloud console of control plane EC2, depending on whichever approach you are taking to run your cluster.
Run the file.
```bash
kubectl apply -f spring-boot-deployment.yml
```

Run the following commands to verify that the deployment was successful and that all components have been created.
```bash
kubectl get pods
kubectl get replicaset
kubectl get deployments
kubectl get servcices
```

Note the External IP address for the Spring Boot app service.  Verify that the Spring Boot app is accessible from this IP when entered as a URL in the address bar of a browswer.

This concludes this part of the exercise.

To continue, change branch to version3.

