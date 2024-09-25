# Actions Runner Controller (ARC)

GitHub Actions automates the deployment of code to different environments, including production. The environments contain the GitHub Runner software which executes the automation. GitHub Runner can be run in GitHub-hosted cloud or self-hosted environments. Self-hosted environments offer more control of hardware, operating system, and software tools. They can be run on physical machines, virtual machines, or in a container. Containerized environments are lightweight, loosely coupled, highly efficient and can be managed centrally. However, they are not straightforward to use.

Actions Runner Controller (ARC) makes it simpler to run self hosted environments on Kubernetes(K8s) cluster.

With ARC you can :

   - Deploy self hosted runners on Kubernetes cluster with a simple set of commands.
   - Auto scale runners based on demand.
   - Setup across GitHub editions including GitHub Enterprise editions and GitHub Enterprise Cloud.

## Getting Started

ARC can be setup with just a few steps.

In this section we will setup prerequisites, deploy ARC into a K8s cluster, and then run GitHub Action workflows on that cluster.

### Installation
By default, actions-runner-controller uses cert-manager for certificate management of Admission Webhook. Make sure you have already installed cert-manager before you install. The installation instructions for the cert-manager can be found below.

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Generate a Personal Access Token (PAT) for ARC to authenticate with GitHub.

  - Login to your GitHub account and Navigate to “Create new Token.”
  - Select repo.
  - Click Generate Token and then copy the token locally ( we’ll need it later).

### Deploy and Configure ARC

#### Helm deployment

```
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm upgrade --install --namespace actions-runner-system --create-namespace --set=authSecret.create=true --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE" --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

#### Kubectl Deployment

```
kubectl apply -f  https://github.com/actions/actions-runner-controller/releases/download/v0.22.0/actions-runner-controller.yaml
kubectl create secret generic controller-manager  -n actions-runner-system  --from-literal=github_token=REPLACE_YOUR_TOKEN_HERE
```
#### Deploying runners with RunnerDeployments

```
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runnerdeploy
spec:
  # This will deploy 2 runners now
  template:
    spec:
      repository: organization/repo-name
      labels:
        - runner-labels-name
```

#### Webhook Driven Scaling
ARC can be configured to scale based on the workflow_job webhook event. The primary benefit of autoscaling on webhooks compared to the pull driven scaling is that ARC is immediately notified of the scaling need.
Webhooks are processed by a separate webhook server. The webhook server receives workflow_job webhook events and scales RunnerDeployments / RunnerSets by updating HRAs configured for the webhook trigger. Below is an example set-up where a HRA has been configured to scale a RunnerDeployment from a workflow_job event:

```
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: runnerdeploy
spec:
  # This will deploy 2 runners now
  template:
    spec:
      repository: organization/repo-name
      labels:
        - runner-labels-name
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: runner-deployment-autoscaler
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: runnerdeploy
  scaleDownDelaySecondsAfterScaleOut: 300
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - organization/repo-name
```


