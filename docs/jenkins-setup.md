# Jenkins on Kubernetes - Complete Setup Guide

## Objective

This guide explains how to deploy a production-style Jenkins setup on Kubernetes.
The setup uses Jenkins as the CI/CD controller, Kubernetes for dynamic build
agents, Kaniko for container image builds, and `kubectl` for application
deployments.

The goal is to build a reusable cloud-native CI/CD platform with:

- Jenkins Controller running inside Kubernetes
- Dynamic Kubernetes build agents
- Kaniko-based Docker image builds without a Docker daemon
- `kubectl` container for Kubernetes deployments
- RBAC-based Kubernetes access
- Docker Hub integration
- Pipeline as Code using a `Jenkinsfile`

---

## Architecture

```text
Developer
   |
   v
GitHub Repository
   |
   v
Jenkins Controller running in Kubernetes
   |
   v
Creates an ephemeral Jenkins agent pod
   |
   v
+---------------+---------------+---------------+
|     jnlp      |    kaniko     |    kubectl    |
+---------------+---------------+---------------+
        |               |               |
        |               |               +--> Deploys to Kubernetes
        |               |
        |               +--> Builds and pushes Docker images
        |
        +--> Connects the agent pod to Jenkins
```

The Jenkins Controller is installed once and remains running in the cluster.
For every pipeline run, Jenkins creates a temporary build pod. That pod contains
the containers needed for the pipeline:

- `jnlp`: connects the build pod back to the Jenkins Controller
- `kaniko`: builds and pushes container images
- `kubectl`: deploys the updated images to Kubernetes

After the pipeline completes, the agent pod is deleted automatically.

---

## Repository Structure

```text
devops-platform/
├── jenkins/
│   ├── values.yaml
│   └── jenkins-rbac.yaml
├── jenkins-agent/
│   └── Dockerfile
├── k8s/
├── monitoring/
└── scripts/
```

---

## Installation Steps

### 1. Create the Jenkins Namespace

Create a dedicated namespace for Jenkins resources:

```bash
kubectl create namespace jenkins
```

---

### 2. Install Jenkins Using Helm

Add the Jenkins Helm repository and update it:

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

Install Jenkins using the custom values file:

```bash
helm install jenkins jenkins/jenkins \
  -n jenkins \
  -f jenkins/values.yaml
```

This installs the Jenkins Controller as a Kubernetes workload using the
configuration defined in `jenkins/values.yaml`.

---

### 3. Apply Jenkins RBAC

Apply the RBAC configuration:

```bash
kubectl apply -f jenkins/jenkins-rbac.yaml
```

This file creates the Kubernetes permissions Jenkins needs for deployment
pipelines, including:

- `ServiceAccount`
- `ClusterRole`
- `ClusterRoleBinding`

These permissions allow Jenkins pipelines to interact with Kubernetes resources
such as deployments, pods, services, and rollout status.

---

### 4. Build the Custom Jenkins Agent Image

Build a custom Jenkins agent image that contains the tools needed by the build
pod.

The custom image should include:

- Jenkins inbound agent support
- Git
- Shell utilities
- Any additional CLI tools required by the pipeline

Example image name:

```text
jenkins-agent:1.0
```

Build and push the image to Docker Hub:

```bash
docker build -t <dockerhub-username>/jenkins-agent:1.0 jenkins-agent/
docker push <dockerhub-username>/jenkins-agent:1.0
```

Use this image in the Jenkins Kubernetes agent configuration or in the
`Jenkinsfile` pod template.

---

### 5. Create the Docker Hub Secret

Kaniko needs Docker Hub credentials so it can push built images to the registry.

Create a Kubernetes secret:

```bash
kubectl create secret generic dockerhub-secret \
  --from-file=config.json=<path-to-docker-config-json> \
  -n jenkins
```

Kaniko expects the Docker config file to be mounted at:

```text
/kaniko/.docker/config.json
```

The `Jenkinsfile` should mount `dockerhub-secret` into the Kaniko container at
that path.

---

### 6. Configure Jenkins Credentials

Configure the required credentials inside Jenkins.

Add credentials for:

- GitHub repository access
- Docker Hub authentication
- Kubernetes access, if required by your Jenkins configuration

These credentials are used by the pipeline to clone code, push Docker images,
and deploy applications.

---

### 7. Configure the Kubernetes Cloud in Jenkins

Configure Jenkins to use Kubernetes as a cloud provider.

In Jenkins:

1. Go to **Manage Jenkins**.
2. Open **Clouds**.
3. Add a new cloud of type **Kubernetes**.
4. Configure the Kubernetes API connection.
5. Set the Jenkins namespace to `jenkins`.
6. Configure the Jenkins service account.

The actual build pod template can be defined directly in the `Jenkinsfile`.
No permanent Jenkins workers are required.

---

## Jenkinsfile Strategy

The pipeline defines the Kubernetes pod template used for each build.

Each pipeline run creates a temporary pod with:

- `jnlp` container for Jenkins connectivity
- `kaniko` container for image builds
- `kubectl` container for Kubernetes deployment commands

The build pod is created at the start of the pipeline and destroyed after the
pipeline finishes.

---

## Build and Deployment Process

### 1. Clone the Repository

Jenkins checks out the application source code from GitHub.

### 2. Build the Backend Image

Kaniko builds the backend Docker image from the backend Dockerfile and pushes it
to Docker Hub.

### 3. Build the Frontend Image

Kaniko builds the frontend Docker image from the frontend Dockerfile and pushes
it to Docker Hub.

### 4. Deploy to Kubernetes

The `kubectl` container updates the Kubernetes deployments with the new image
tags.

Example:

```bash
kubectl set image deployment/backend backend=<dockerhub-username>/backend:<tag>
kubectl set image deployment/frontend frontend=<dockerhub-username>/frontend:<tag>
```

Kubernetes then performs a rolling update automatically.

Check rollout status:

```bash
kubectl rollout status deployment/backend
kubectl rollout status deployment/frontend
```

---

## Why Use Kaniko?

Traditional Jenkins Docker builds usually require access to a Docker daemon:

```text
Jenkins
   |
   v
Docker daemon
   |
   v
docker build
```

This often requires mounting the Docker socket into the Jenkins agent, which can
create security risks.

Common problems with Docker socket builds:

- Requires Docker daemon access
- Often needs privileged containers
- Increases cluster security risk
- Makes builds less Kubernetes-native

Kaniko solves this by building images directly from a Dockerfile without a
Docker daemon:

```text
Kaniko
   |
   v
Reads Dockerfile
   |
   v
Builds image
   |
   v
Pushes image to registry
```

Kaniko is a better fit for Kubernetes CI/CD because it is daemonless,
container-native, and does not require privileged Docker access.

---

## Dynamic Agent Lifecycle

```text
Pipeline starts
   |
   v
Jenkins creates agent pod
   |
   v
Pod pending
   |
   v
Containers creating
   |
   v
Pod running
   |
   v
Pipeline executes
   |
   v
Pipeline completes
   |
   v
Agent pod terminates
   |
   v
Agent pod deleted
```

Because the agents are ephemeral, no idle workers remain after a build. This
keeps the setup cleaner and allows Jenkins builds to scale with Kubernetes
capacity.

---

## Common Issues and Fixes

### Jenkins Is Stuck After a Machine Restart

Sometimes the old StatefulSet pod can become stale after a restart.

Fix:

```bash
kubectl delete pod jenkins-0 -n jenkins
```

The StatefulSet recreates the Jenkins pod automatically.

---

### Dynamic Agent Never Starts

Check the following:

- Kubernetes Cloud configuration in Jenkins
- Jenkins service account
- RBAC permissions
- Agent image name and tag
- Image pull permissions
- Pod template configuration in the `Jenkinsfile`

Useful command:

```bash
kubectl get pods -n jenkins
kubectl describe pod <agent-pod-name> -n jenkins
```

---

### Kaniko Authentication Fails

Verify that the Docker Hub secret exists:

```bash
kubectl get secret dockerhub-secret -n jenkins
```

Also verify that the secret is mounted into the Kaniko container at:

```text
/kaniko/.docker/config.json
```

If the mount path is wrong, Kaniko will not be able to authenticate with Docker
Hub.

---

### `kubectl` Permission Denied

This usually means Jenkins does not have the required Kubernetes RBAC
permissions.

Apply the RBAC file again:

```bash
kubectl apply -f jenkins/jenkins-rbac.yaml
```

Then verify the service account permissions:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:jenkins:jenkins
```

---

### Deployment Update Fails

Check that the image exists in Docker Hub and that Kubernetes can pull it.

Useful commands:

```bash
kubectl rollout status deployment/<deployment-name>
kubectl describe deployment <deployment-name>
kubectl describe pod <pod-name>
```

Also verify the deployment container name before using `kubectl set image`.

---

## Why This Architecture?

This architecture is useful because it provides:

- No Docker daemon dependency
- No static Jenkins workers
- Kubernetes-native build execution
- Secure image builds
- Elastic build agents
- Automatic cleanup after builds
- Easy horizontal scaling
- A reusable CI/CD pattern for multiple applications

---

## Future Enhancements

Possible improvements:

- GitHub webhooks
- Argo CD GitOps deployment
- Trivy security scanning
- SonarQube code quality checks
- Slack notifications
- Blue-green deployments
- Canary deployments
- Policy as Code
- Terraform infrastructure provisioning
- Crossplane infrastructure management
- Backstage developer portal integration

---

## Final Flow

```text
Developer
   |
   v
Git push
   |
   v
GitHub
   |
   v
Jenkins Controller
   |
   v
Creates Kubernetes agent pod
   |
   v
jnlp + kaniko + kubectl
   |
   v
Build images
   |
   v
Push images to Docker Hub
   |
   v
kubectl updates Kubernetes deployments
   |
   v
Rolling deployment
   |
   v
Application updated
```

This setup provides a reusable cloud-native CI/CD platform. For a new
Kubernetes application, the same Jenkins architecture can be reused by changing
the repository, image names, Dockerfiles, and deployment manifests.
