# Kubernetes deployments
This repo explains some key deployment strategies using Docker, Kubernetes, AWS 

## Blue-Green Deployment with Kubernetes, AWS, and Docker
Blue-Green Deployment is a software release management strategy that reduces downtime and risk by running two identical environments, referred to as Blue and Green. 
One environment (let’s say Blue) is the live, production environment, and the other (Green) is used for testing the new release.

In the context of Kubernetes, AWS, and Docker, this strategy can be achieved by using 
- Kubernetes clusters to manage containers
- Docker images to build the application and
- AWS services (like Elastic Load Balancer, Elastic Kubernetes Service, etc.) to manage deployment and traffic routing.

### Step-by-Step Guide

#### Step 1: Create Kubernetes Cluster on AWS (EKS)
Using `eksctl`, we can easily create a Kubernetes cluster on AWS.

##### Create a Cluster
Run the following command to create the cluster:

```
eksctl create cluster --name blue-green-cluster
                      --region us-west-2
                      --nodegroup-name standard-workers
                      --node-type t2.micro
                      --nodes 2
```

##### Configure kubectl to interact with the newly created cluster
```
aws eks --region us-west-2 update-kubeconfig --name blue-green-cluster
```

##### Now you can verify your cluster by running
```
kubectl get nodes
```

#### Step 2: Dockerize the Application

Create a Dockerfile to containerize the application

```
FROM node:14

WORKDIR /app
COPY . /app

RUN npm install

EXPOSE 8080

CMD ["node", "app.js"]
```

Build the Docker image:

```
docker build -t my-app:v1 .
```

Push the image to Docker Hub (or AWS ECR)

```
docker tag my-app:v1 yourdockerhubusername/my-app:v1
docker push yourdockerhubusername/my-app:v1
```

#### Step 3: Deploy Blue Environment
Now that we have a Docker image, we can deploy it to Kubernetes.

Create a Kubernetes Deployment for Blue Environment (blue-deployment.yaml):

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
        - name: my-app
          image: yourdockerhubusername/my-app:v1
          ports:
            - containerPort: 8080
```

Create a Service to Expose Blue Environment (blue-service.yaml):

```
apiVersion: v1
kind: Service
metadata:
  name: my-app-blue
spec:
  selector:
    app: my-app
    version: blue
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer

```

Apply the Blue Deployment and Service:

```
kubectl apply -f blue-deployment.yaml
kubectl apply -f blue-service.yaml
```

This will create the Blue environment and expose it through a LoadBalancer.

#### Step 4: Deploy Green Environment

Create a Kubernetes Deployment for Green Environment (green-deployment.yaml):

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
        - name: my-app
          image: yourdockerhubusername/my-app:v2
          ports:
            - containerPort: 8080
```


Create a Service to Expose Green Environment (green-service.yaml):

```
apiVersion: v1
kind: Service
metadata:
  name: my-app-green
spec:
  selector:
    app: my-app
    version: green
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

Apply the Green Deployment and Service:

```
kubectl apply -f green-deployment.yaml
kubectl apply -f green-service.yaml
```

#### Step 5: Switch Traffic Between Blue and Green

At this point, both the Blue and Green environments are deployed in the cluster, 
but we need to direct traffic to one of them.

**Method 1**: Switch Traffic Using Kubernetes Services
1. Initially, the Blue environment should be exposed:

```
kubectl expose deployment my-app-blue --type=LoadBalancer --name=my-app
```
2. When you're ready to switch to the Green environment, change the service to point to the Green deployment:

```
kubectl expose deployment my-app-green --type=LoadBalancer --name=my-app
```

**Method 2**: Using AWS Route 53 for Traffic Switching

You can also manage traffic routing between Blue and Green environments using AWS Route 53 by pointing 
a domain name (e.g., myapp.com) to the LoadBalancer IPs for both the Blue and Green environments and then 
routing traffic between them based on weights.

#### Step 6: Rollback (If Necessary)
If there are any issues with the Green environment, you can quickly roll back to the Blue environment by 
modifying the service again or using Route 53 to shift traffic back to Blue.

**Conclusion**

This tutorial demonstrated how to set up a Blue-Green Deployment using Kubernetes, AWS, and Docker. 
By following this pattern, you can ensure zero-downtime deployments and quickly revert to a stable version of your application if issues arise.

































