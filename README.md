## Kubernetes for ML Engineers

#### Table of contents

- [What is this repo about?](#what-is-this-repo-about)
- [Steps](#steps)
    - [1. Install the tools](#1-install-the-tools)
    - [2. Create a local Kubernetes cluster](#2-create-a-local-kubernetes-cluster)
    - [3. Write the business logic of your app](#3-write-the-business-logic-of-your-app)
    - [4. Containerize your app with Docker](#4-containerize-your-app-with-docker)
    - [5. Build the Docker image and run it locally](#5-build-the-docker-image-and-run-it-locally-optional)
    - [6. Push the Docker image to the local Kubernetes cluster](#6-push-the-docker-image-to-the-local-kubernetes-cluster)
    - [7. Deploy the app as a Kubernetes service](#7-deploy-the-app-as-a-kubernetes-service)
    - [8. Test it works](#8-test-it-works)
    - [9. Run the whole thing in one go](#9-run-the-whole-thing-in-one-go)
- [Wanna learn more Real World ML/MLOps?](#wanna-learn-more-real-world-mlmlops)


### What is this repo about?
This is a step by step guide to help you understand the basics of Kubernetes.

You will learn how to build and deploy a containerized app (in this case, a simple FastAPI app)
to a Kubernetes cluster.

Because I want you to become a Real World ML/MLOps Ninja.

Let's get started.

### Steps

### 1. Install the tools

- [uv](https://docs.astral.sh/uv/getting-started/installation/) to create the project and manage the dependencies.
- [Docker](https://docs.docker.com/get-docker/) to build and run docker images.
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) to create a local Kubernetes cluster.
- [kubectl](https://kubernetes.io/docs/tasks/tools/) to interact with the Kubernetes cluster.

### 2. Create a local Kubernetes cluster

We will use `kind` to create a local Kubernetes cluster. It will be a simple cluster that
will run entirely on your machine, using as Kubernetes nodes simple Docker containers.

A local cluster like the one we are creating here is useful for development and CI pipelines,
where you need a minimal cluster to run integration tests for your applications.

> **What about production-ready clusters?**
> 
> Production clusters typically consist of multiple nodes, running in a cloud provider, or
> a private data center.
>
> Production cluster creation, configuration, and maintenance is something you won't be doing in your day to day as ML Engineer. This is something that you either:
> 
> - pay your cloud provider to do it for you, like when you use AWS EKS, Google GKE, Azure AKS, etc
> or
> - hire a top-notch engineer like **[Marius](https://lluminarr.com/)** to do it for you, so
> you get a solution that is cost-effective and scalable.

We will create a cluster consisting of
- 1 control plane node -> where the core Kubernetes components run
- 2 worker nodes -> where the apps we will deploy will run.

The configuration file for the cluster is the following:

```yaml
# kind.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "CriticalAddonsOnly=true,eks-k8s-version=1.29"

  - role: worker
    kubeadmConfigPatches:
      - |
        kind: JoinConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "CriticalAddonsOnly=true,eks-k8s-version=1.29"

  - role: worker
    labels:
      "CriticalAddonsOnly": "true"
      "eks-k8s-version": "1.29"
```

Create the cluster with the name you want (e.g. `cluster-123`) using the above configuration:
```bash
kind create cluster --config kind.yaml --name cluster-123
```

Set the kubectl context to the cluster we just created, so you can interact with the cluster using `kubectl`:

```bash
kubectl config use-context kind-cluster-123
```

Get the list of nodes in the cluster:

```bash
kubectl get nodes

NAME                        STATUS   ROLES           AGE   VERSION
cluster-123-control-plane   Ready    control-plane   15m   v1.32.2
cluster-123-worker          Ready    <none>          14m   v1.32.2
cluster-123-worker2         Ready    <none>          14m   v1.32.2
```

Voila! You have a local Kubernetes cluster running on your machine.

Let's now move on to the ML engineering work.

#### 3. Write the business logic of your app

In this case, we will create a simple FastAPI app that returns the current time when you hit the `/health` endpoint.

We will use `uv` to create the project, which is the most ergonomic way to create and package your Python code.

1. Create the boilerplate code with:
    ```bash
    uv new simple-api
    ```

2. Add FastAPI to the project:
    ```bash
    uv add fastapi[standard]
    ```

3. Rename the `hello.py` file to `api.py` and copy this code:
    ```python
    from fastapi import FastAPI
    from datetime import datetime

    app = FastAPI()

    @app.get('/health')
    async def health():
        return {
            'status': 'healthy',
            'timestamp': datetime.now().isoformat()
        }
    ```

Feel free to adjust this code to your needs.

#### 4. Containerize your app with Docker

We write a [multi-stage Dockerfile](./Dockerfile) to reduce the final image size.

It has 2 stages:

- builder -> where we install the project dependencies with `uv` and copy the code
- runner -> where we run the FastAPI app


#### 5. Build the Docker image and run it locally (optional)

To build the image, run the following command:

```bash
docker build -t simple-api:v1.0.0 .
```

And to run it locally, run the following command:

```bash
docker run -it -p 5005:5000 simple-api:v1.0.0
```
Observe how we forward the container's port 5000 to the host's port 5005.

At this point, you should be able to hit the `/health` endpoint at `http://localhost:5005/health` and get the current time.

```bash
curl http://localhost:5005/health
```

Congratulations! You have just built and run a Docker container locally.

Let's now take things to the next level and run it in a Kubernetes cluster.


#### 6. Push the Docker image to the local Kubernetes cluster

Before we can deploy our app to the cluster, we need to push the Docker image to the local Kubernetes cluster.

To do that, we will use the `kind` CLI to load the image into the cluster.

```bash
kind load docker-image simple-api:v1.0.0 --name cluster-123
```

#### 7. Deploy the app as a Kubernetes service

Now that we have the image in the cluster, we can deploy the app as a Kubernetes service.

We will need to create 2 resources:

- a [deployment.yaml](./deployment.yaml) -> which will define the pods that will run the app. In our case, we will have 3 replicas of the app.
- a [service.yaml](./service.yaml) -> which will define how to access the app from outside the cluster

Don't worry about the manifests for now. Kubernetes YAML files are notoriously verbose and hard to read.
And if you are scared of them, you are not alone. I am scared of them too.

To deploy the app, we use the `kubectl` CLI to apply the Kubernetes manifests:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

You can check the status of the deployment with:

```bash
kubectl get pods

NAME                          READY   STATUS    RESTARTS   AGE
simple-api-7f4bbc478b-f7wdx   1/1     Running   0          5m26s
simple-api-7f4bbc478b-fjx2m   1/1     Running   0          5m26s
simple-api-7f4bbc478b-gfntx   1/1     Running   0          5m26s
```


#### 8. Test it works

To test that the app is working, we can use the `kubectl` CLI to port-forward the service to our local machine:

```bash
kubectl port-forward svc/simple-api 5005:5000
```

And then we can hit the `/health` endpoint at `http://localhost:5005/health` and get the current time.

```bash
curl http://localhost:5005/health

{"status":"healthy","timestamp":"2025-02-21T15:25:55.445524"}
```

#### 9. Run the whole thing in one go

```bash
# Create the Kubernetes cluster
make cluster

# Deploy the FastAPI app to the cluster
make deploy

# Test connecting to the app
make test
```

CONGRATULATIONS! You have just deployed a FastAPI app to a local Kubernetes cluster.

The sky is the limit from here.

### Wanna learn more Real World ML/MLOps?

Subscribe for free to my newsletter to get notified when I publish new articles and courses:
### [👉👉🏻👉🏼👉🏽👉🏾👉🏿 Subscribe](https://paulabartabajo.substack.com/)

### [👉👉🏻👉🏼👉🏽👉🏾👉🏿 Courses](https://realworldml.net/courses)
