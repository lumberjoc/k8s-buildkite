# k8s-buildkite
How-To guide for setting up a Buildkite agent in your Kubernetes cluster. I went through a few different tutorials explaining how to setup Buildkite, but was unable to find a tutorial I liked on how to setup an agent on a Kubernetes cluster. As a result, I've created this straight forward tutorial on how to setup a Buildkite agent. 

Disclaimer: This is not production ready, but it should be good exercise to get you familiarized with Buildkite, k8s networking, rbac, and more! Hope you enjoy!

## File & Directory Breakdown
```
k8s-buildkite
├── README.md
└── buildkite-agent
    ├── deploy
    │   ├── 0-deployment.yaml
    │   ├── 1-service.yaml
    │   ├── 2-ingress.yaml
    └── rbac
        ├── 0-clusterrole.yaml
        ├── 1-clusterrolebinding.yaml
        └── 2-serviceaccount.yaml
```
- `buildkite-agent/`: This directory contains the configuration files for the Buildkite agent.
  - `deploy/`: This directory contains the Kubernetes deployment files for the Buildkite agent.
    - `0-deployment.yaml`: This file defines the Kubernetes Deployment for the Buildkite agent, specifying the container image, environment variables, and resource requests/limits.
    - `1-service.yaml`: This file defines the Kubernetes Service to expose the Buildkite agent.
    - `2-ingress.yaml`: This file defines the Kubernetes Ingress to expose the Buildkite agent to external traffic.
    - [Coming Soon]`3-hpa.yaml`: This file defines the Horizontal Pod Autoscaler (HPA) for the Buildkite agent, which automatically scales the number of agent pods based on CPU utilization.
  - `rbac/`: This directory contains the RBAC configuration files for the Buildkite agent.
    - `0-clusterrole.yaml`: This file defines a ClusterRole with the necessary permissions for the Buildkite agent.
    - `1-clusterrolebinding.yaml`: This file creates a ClusterRoleBinding to bind the ClusterRole to the Buildkite agent's ServiceAccount.
    - `2-serviceaccount.yaml`: This file creates the ServiceAccount for the Buildkite agent.


## Step 1: Set up Kubernetes Cluster

We'll be using minikube for this tutorial since it's a lightweight and easy-to-use Kubernetes cluster. You can install minikube by following the instructions for your operating system from the official website: https://minikube.sigs.k8s.io/docs/start/

Once installed, start the minikube cluster with the following command:
```
minikube start
```

## Step 2: Create a Namespace

Namespaces in Kubernetes provide a way to logically partition resources within the same cluster. Let's create a namespace for our "hello-world" application:
```
kubectl create namespace buildkite
```

## Step 3: Configure RBAC

RBAC (Role-Based Access Control) is a crucial aspect of Kubernetes security. It defines the permissions for users or services to access and perform operations on Kubernetes resources.

First, create a ServiceAccount for the Buildkite agent:
```
# buildkite 0-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: buildkite-agent
  namespace: buildkite
```

Apply the ServiceAccount:
```
kubectl apply -f 0-serviceaccount.yaml -n buildkite
```

Next, create a ClusterRole that grants the necessary permissions for the Buildkite agent:
```
# buildkite 1-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: buildkite-agent-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Apply the ClusterRole:
```
kubectl apply -f 1-clusterrole.yaml -n buildkite
```

Finally, create a ClusterRoleBinding to bind the ClusterRole to the ServiceAccount:
```
# buildkite 2-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: buildkite-agent-rolebinding
subjects:
- kind: ServiceAccount
  name: buildkite-agent
  namespace: buildkite
roleRef:
  kind: ClusterRole
  name: buildkite-agent-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the ClusterRoleBinding:
```
kubectl apply -f 2-clusterrolebinding.yaml
```

## Step 4: Set up Buildkite Agent

We'll be using Buildkite's agent to run our pipelines. You can sign up for a Buildkite account and follow their instructions to set up the agent token.
Once you have the Buildkite agent token, you'll need to create a Kubernetes deployment that uses the agent-token and a service (don't apply immediately after creating files):
```
# buildkite-agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: buildkite-agent
  namespace: buildkite
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
        - name: DOCKER_HUB_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: dockerhub-credentials
              key: access-token
        - name: BUILDKITE_UNCLAIM_CONTAINERS_VIA_API
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_CPU_LIMIT
          value: "4"
        - name: BUILDKITE_AGENT_META_DATA_MEMORY_LIMIT
          value: "8Gi"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_NAMESPACE
          value: "buildkite"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

```
# buildkite-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: buildkite-agent
  namespace: buildkite
spec:
  selector:
    app: buildkite-agent
  ports:
  - port: 8000
    targetPort: 8000
```

Before applying, you'll need to create a Kubernetes secret for the Buildkite agent token and your Dockerhub access token:
1) Agent-Token
```
kubectl create secret generic buildkite-agent-token --from-literal=token=YOUR_BUILDKITE_AGENT_TOKEN -n buildkite
```
Replace YOUR_BUILDKITE_AGENT_TOKEN with the actual token from your Buildkite account.

2) [SKIP] Dockerhub-Token
```
kubectl create secret generic dockerhub-credentials --from-literal=access-token=YOUR_DOCKERHUB_ACCESS_TOKEN -n buildkite
```
Replace YOUR_DOCKERHUB_ACCESS_TOKEN with the actual token from your Buildkite account.


Before deploying the Buildkite agent, you need to create the "kubernetes" agent-queue in your Buildkite account. This can be done through the Buildkite web interface or using the Buildkite CLI. If you don't do this your agent will not connect. 

Apply the deployment:
```
kubectl apply -f 0-deployment.yaml -n buildkite
```

Apply the service:
```
kubectl apply -f 1-service.yaml -n buildkite
```

## [COMING SOON: Skip for now] Step 5: Create a Horizontal Pod Autoscaler (HPA)

If you anticipate varying workloads or want to automatically scale the number of agent pods based on resource utilization, implementing an HPA is a good idea.

Create a new file `3-hpa.yaml` in the `k8s-buildkite/buildkite-agent/deploy` directory:
```
# k8s-buildkite/buildkite-agent/deploy/3-hpa.yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: buildkite-agent-hpa
  namespace: buildkite
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: buildkite-agent
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```

This HPA configuration will automatically scale the number of Buildkite agent pods between 1 and 5, based on the average CPU utilization. When the average CPU utilization reaches 50%, the HPA will start scaling up the number of pods.

Apply the HPA:
```
kubectl apply -f k8s-buildkite/buildkite-agent/deploy/3-hpa.yaml
```

Update the `0-deployment.yaml` in the `k8s-buildkite/buildkite-agent/deploy` directory to include resource requests and limits for the Buildkite agent pods:
```
# k8s-buildkite/buildkite-agent/deploy/0-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: buildkite-agent
  namespace: buildkite
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
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        env:
        - name: BUILDKITE_AGENT_TOKEN
          valueFrom:
            secretKeyRef:
              name: buildkite-agent-token
              key: token
        - name: DOCKER_HUB_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: dockerhub-credentials
              key: access-token
        - name: BUILDKITE_UNCLAIM_CONTAINERS_VIA_API
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_CPU_LIMIT
          value: "4"
        - name: BUILDKITE_AGENT_META_DATA_MEMORY_LIMIT
          value: "8Gi"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES
          value: "true"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_NAMESPACE
          value: "buildkite"
        - name: BUILDKITE_AGENT_META_DATA_KUBERNETES_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

Apply the updated deployment:
```
kubectl apply -f k8s-buildkite/buildkite-agent/deploy/0-deployment.yaml
```

## Step 6: Set up Ingress/Egress

Since the Buildkite agent needs to communicate with the Buildkite service, we need to configure networking appropriately. In this case, we'll use an Ingress resource to expose the Buildkite agent service to the internet:
```
# buildkite-agent-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: buildkite-agent-ingress
  namespace: buildkite
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
kubectl apply -f 2-ingress.yaml -n buildkite
```

Post-Deploy Failed Connection Logs:
```
| 2024-04-08 17:36:46 INFO   Registering agent with Buildkite...
│ 2024-04-08 17:36:46 WARN   Failed to find unique machine-id: machineid: machineid: open /etc/machine-id: no such file or directory
│ 2024-04-08 17:36:46 WARN   POST https://agent.buildkite.com/v3/register: 400 Bad Request: Queue is required when registering agents to a cluster (Attempt 1/30 Retrying
```

Post-Deploy Successful Connection Logs:
```
| 2024-04-08 17:36:46 INFO   Registering agent with Buildkite...
| 2024-04-08 17:38:07 INFO   Successfully registered agent "buildkite-agent-687bb4f685-kqx7b-1" with tags [queue=kubernetes]                                             
│ 2024-04-08 17:38:07 INFO   Starting 1 Agent(s)                                                                                                                         
│ 2024-04-08 17:38:07 INFO   You can press Ctrl-C to stop the agents                                                                                                      
│ 2024-04-08 17:38:07 INFO   buildkite-agent-687bb4f685-kqx7b-1 Connecting to Buildkite...                                                                                
│ 2024-04-08 17:38:07 INFO   buildkite-agent-687bb4f685-kqx7b-1 Waiting for work...
```

## Create a pipeline and run a job

Your agent should now be connected to the "kubernetes" queue in Buildkite. Feel free to start making pipelines and building until your hearts content. I will be updating this repo with HPA mechanisms as well as a simple docker build/deploy pipeline in the future. Feel free to checkout Buildkites [Getting Started](https://buildkite.com/docs/tutorials/getting-started) guide to gain more familiarity with the tool. Thanks for reading!

