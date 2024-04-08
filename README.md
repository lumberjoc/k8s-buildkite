# k8s-buildkite
How to guide for setting up a simple Buildkite pipeline on your Kubernetes cluster. I went through a few different tutorials explaining how to setup Buildkite, but was unable to find an adequate tutorial on how to setup a pipeline on a Kubernetes cluster. As a result, I've created this straight forward tutorial on how to setup a simple pipeline. Hope you enjoy!

## File & Directory Breakdown
```
hello-world-app/
├── .buildkite
│   └── pipeline.yml
├── buildkite-agent
│   ├── buildkite-clusterrole.yaml
│   ├── buildkite-clusterrolebinding.yaml
│   └── buildkite-serviceaccount.yaml
├── Dockerfile
├── deployment.yaml
├── main.go
└── service.yaml
```
The buildkite-agent directory contains the RBAC configuration files for the Buildkite agent:
- `buildkite-clusterrole.yaml`: This file defines a ClusterRole with the necessary permissions for the Buildkite agent.
- `buildkite-clusterrolebinding.yaml`: This file creates a ClusterRoleBinding to bind the ClusterRole to the Buildkite agent's ServiceAccount.
- `buildkite-serviceaccount.yaml`: This file creates the ServiceAccount for the Buildkite agent.

- `hello-world-app/`: This is the root directory for your "Hello World" application.
- `.buildkite/`: This directory contains the Buildkite pipeline configuration.
  - `pipeline.yml`: This file defines the steps in your CI/CD pipeline, such as building the Docker image, pushing it to a registry, and deploying it to Kubernetes.
- `Dockerfile`: This file contains the instructions for building the Docker image for your Go application.
- `deployment.yaml`: This Kubernetes manifest file defines the deployment for your "Hello World" application, specifying the number of replicas, container image, and other configurations.
- `service.yaml`: This Kubernetes manifest file defines a service to expose your "Hello World" application deployment to other pods or external traffic.
- `main.go`: This is the source code file for your simple "Hello World" Go application.


## Step 1: Set up Kubernetes Cluster

We'll be using minikube for this tutorial since it's a lightweight and easy-to-use Kubernetes cluster. You can install minikube by following the instructions for your operating system from the official website: https://minikube.sigs.k8s.io/docs/start/

Once installed, start the minikube cluster with the following command:
```
minikube start
```

## Step 2: Create a Namespace

Namespaces in Kubernetes provide a way to logically partition resources within the same cluster. Let's create a namespace for our "hello-world" application:
```
kubectl create namespace hello-world
```

## Step 3: Configure RBAC

RBAC (Role-Based Access Control) is a crucial aspect of Kubernetes security. It defines the permissions for users or services to access and perform operations on Kubernetes resources.

First, create a ServiceAccount for the Buildkite agent:
```
# buildkite-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: buildkite-agent
  namespace: hello-world
```

Apply the ServiceAccount:
```
kubectl apply -f buildkite-serviceaccount.yaml -n hello-world
```

Next, create a ClusterRole that grants the necessary permissions for the Buildkite agent:
```
# buildkite-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: buildkite-agent-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Finally, create a ClusterRoleBinding to bind the ClusterRole to the ServiceAccount:
```
# buildkite-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: buildkite-agent-rolebinding
subjects:
- kind: ServiceAccount
  name: buildkite-agent
  namespace: hello-world
roleRef:
  kind: ClusterRole
  name: buildkite-agent-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the ClusterRoleBinding:
```
kubectl apply -f buildkite-clusterrolebinding.yaml
```

## Step 4: Set up Buildkite Agent

We'll be using Buildkite's agent to run our pipelines. You can sign up for a Buildkite account and follow their instructions to set up the agent.
Once you have the Buildkite agent configured, you'll need to create a Kubernetes deployment and service for it:
```
# buildkite-agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: buildkite-agent
  namespace: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: buildkite-agent
  template:
    metadata:
      labels:
        app: buildkite-agent
    spec:
      serviceAccountName: buildkite-agent
      containers:
      - name: buildkite-agent
        image: buildkite/agent:latest
        env:
        - name: BUILDKITE_AGENT_TOKEN
          valueFrom:
            secretKeyRef:
              name: buildkite-agent-token
              key: token
        - name: BUILDKITE_UNCLAIM_CONTAINERS_VIA_API
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_CPU_LIMIT
          value: "4"
        - name: BUILDKITE_AGENT_META_DATA_MEMORY_LIMIT
          value: "8Gi"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_NAMESPACE
          value: "hello-world"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
---
apiVersion: v1
kind: Service
metadata:
  name: buildkite-agent
  namespace: hello-world
spec:
  selector:
    app: buildkite-agent
  ports:
  - port: 8000
    targetPort: 8000
```

You'll need to create a Kubernetes secret for the Buildkite agent token:
```
kubectl create secret generic buildkite-agent-token --from-literal=token=YOUR_BUILDKITE_AGENT_TOKEN -n hello-world
```
Replace YOUR_BUILDKITE_AGENT_TOKEN with the actual token from your Buildkite account.

Before deploying the Buildkite agent, you need to create the "kubernetes" agent queue in your Buildkite account. This can be done through the Buildkite web interface or using the Buildkite CLI. If you don't do this your agent will not connect. 

Failed Connection Log:
```
| 2024-04-08 17:36:46 INFO   Registering agent with Buildkite...
│ 2024-04-08 17:36:46 WARN   Failed to find unique machine-id: machineid: machineid: open /etc/machine-id: no such file or directory
│ 2024-04-08 17:36:46 WARN   POST https://agent.buildkite.com/v3/register: 400 Bad Request: Queue is required when registering agents to a cluster (Attempt 1/30 Retrying
```

Successful Connection Log:
```
| 2024-04-08 17:36:46 INFO   Registering agent with Buildkite...
| 2024-04-08 17:38:07 INFO   Successfully registered agent "buildkite-agent-687bb4f685-kqx7b-1" with tags [queue=kubernetes]                                             
│ 2024-04-08 17:38:07 INFO   Starting 1 Agent(s)                                                                                                                         
│ 2024-04-08 17:38:07 INFO   You can press Ctrl-C to stop the agents                                                                                                      
│ 2024-04-08 17:38:07 INFO   buildkite-agent-687bb4f685-kqx7b-1 Connecting to Buildkite...                                                                                
│ 2024-04-08 17:38:07 INFO   buildkite-agent-687bb4f685-kqx7b-1 Waiting for work...
```


Apply the deployment and service:
```
kubectl apply -f buildkite-agent-deployment.yaml -n hello-world
```

Step 5: Set up Ingress/Egress

Since the Buildkite agent needs to communicate with the Buildkite service, we need to configure networking appropriately. In this case, we'll use an Ingress resource to expose the Buildkite agent service to the internet:
```
# buildkite-agent-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: buildkite-agent-ingress
  namespace: hello-world
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: buildkite-agent
            port:
              number: 8000
```

Apply the Ingress:
```
kubectl apply -f buildkite-agent-ingress.yaml -n hello-world
```

## Step 6: Set up "Hello World" Application

Now, let's create a simple "Hello World" application that we'll deploy to Kubernetes and set up a CI/CD pipeline for.

Create a new directory for your application:
```
mkdir hello-world-app
cd hello-world-app
```

Create a simple main.go file:
```
// main.go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    fmt.Println("Server listening on port 8080")
    http.ListenAndServe(":8080", nil)
}
```

Create a Dockerfile to build the Go application:
```
# Dockerfile
FROM golang:1.19-alpine

WORKDIR /app

COPY . .

RUN go build -o hello-world .

EXPOSE 8080

CMD ["./hello-world"]
```

Create a deployment.yaml file to deploy the application to Kubernetes:
```
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: hello-world:latest
        ports:
        - containerPort: 8080
```

Create a service.yaml file to expose the deployment:
```
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 8080
```

## Step 7: Set up CI/CD Pipeline

Now, let's set up a CI/CD pipeline using Buildkite to build and deploy our "Hello World" application.

Create a .buildkite directory in your project:
```
mkdir .buildkite
```

Inside the .buildkite directory, create a pipeline.yml file with the following content:
```
# pipeline.yml
steps:
  - label: "Build and Push Docker Image"
    commands:
      - docker build -t hello-world:$BUILDKITE_BUILD_NUMBER .
      - docker push hello-world:$BUILDKITE_BUILD_NUMBER

  - wait

  - label: "Deploy to Kubernetes"
    commands:
      - kubectl set image deployment/hello-world hello-world=hello-world:$BUILDKITE_BUILD_NUMBER -n hello-world
    agents:
      queue: "kubernetes"

# Configure the Kubernetes queue
agent-queues:
  - "kubernetes"

# Configure the queue settings
queue-configuration:
  kubernetes:
    cluster: "minikube"
    namespace: "hello-world"
    agent-service-account: "buildkite-agent"
```

This pipeline defines two steps:

1. Build and Push Docker Image: This step builds a Docker image for your "Hello World" application and pushes it to a Docker registry (you'll need to configure a registry and replace hello-world with your registry URL).
2. Deploy to Kubernetes: This step updates the hello-world deployment in the hello-world namespace with the new Docker image built in the previous step.

The agents section specifies that the "Deploy to Kubernetes" step should be run by an agent in the "kubernetes" queue.

The agent-queues section defines the "kubernetes" queue, which will be used by the Buildkite agents running in your Kubernetes cluster.

The queue-configuration section configures the "kubernetes" queue to use the minikube cluster, the hello-world namespace, and the buildkite-agent ServiceAccount.


## Step 8: Trigger the Pipeline

Now that you have the pipeline defined, you can trigger it from the Buildkite web interface or using the Buildkite CLI.

After triggering the pipeline, you should see the steps executing in the Buildkite web interface. Once the pipeline completes successfully, your "Hello World" application will be deployed to the hello-world namespace in your Kubernetes cluster.

You can verify the deployment by running:
```
kubectl get pods -n hello-world
```

You should see three pods running your "Hello World" application.

To access the application, you can use the minikube service command:
```
minikube service hello-world -n hello-world
```

This will open a browser window or provide you with the URL to access your "Hello World" application.

And that's it! You've successfully set up a simple CI/CD pipeline for a "Hello World" application using Kubernetes and Buildkite.

This tutorial covered several key concepts, including:
- Setting up a Kubernetes cluster
- Creating namespaces and configuring RBAC
- Deploying and configuring the Buildkite agent
- Setting up networking (Ingress)
- Creating a simple Go application
- Defining a CI/CD pipeline with Buildkite
- Triggering the pipeline and verifying the deployment

Remember, this is a basic example, and in a production environment, you'd want to consider additional aspects like persistent storage, more robust networking configurations, monitoring, and more advanced pipeline steps (e.g., running tests, static code analysis, etc.).
