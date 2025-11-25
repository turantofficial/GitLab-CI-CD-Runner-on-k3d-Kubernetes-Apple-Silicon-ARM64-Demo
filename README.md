GitLab CI/CD Runner on k3d Kubernetes (Apple Silicon ARM64)

This repository shows how to run a fully working GitLab Runner inside a k3d Kubernetes cluster on a Mac M1/M2/M3 (ARM64).

It solves the typical ARM64 issues:

- exec format error

- GitLab helper image fails (no match for platform in manifest)

- Kubernetes RBAC Forbidden

- init-permissions container fails

- jobs stuck in Preparing environment

This README guides you through:

- Installing k3d  
- Creating a local Kubernetes cluster  
- Deploying a GitLab Runner via Helm  
- Fixing helper image architecture  
- Fixing RBAC permissions  
- Running a working GitLab CI/CD job  

What is k3d?

k3d runs the lightweight Kubernetes distribution k3s inside Docker.
This makes Kubernetes extremely fast, lightweight, and ARM-native.

Feature	k3d
Speed	Starts in seconds
Resource usage Very low (great for 8GB RAM Macs)
Architecture	Fully ARM64 compatible
Use cases	CI/CD, DevOps learning, GitOps, Helm, CKA practice
Prerequisites
```
brew install k3d kubectl helm
```

Ensure Docker Desktop or Rancher Desktop is running.

1. Create k3d Kubernetes Cluster
```
k3d cluster create dev \
  --servers 1 \
  --agents 1 \
  --port "8080:80@loadbalancer"
```

Verify:
```
kubectl get nodes
```
2. Create Namespace & Kubeconfig Secret
```
kubectl create namespace gitlab-runner
```
```
kubectl create secret generic k3d-kubeconfig \
  -n gitlab-runner \
  --from-file=config=$HOME/.kube/config
```

3. Create values.yaml (ARM64 working version)

Create:

values.yaml


Paste:
```
gitlabUrl: https://gitlab.com
runnerRegistrationToken: "YOUR_GITLAB_GLRT_TOKEN"

rbac:
  create: true

runners:
  helpers:
    image: "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:arm64-v18.6.1"

  config: |
    [[runners]]
      name = "k3d-arm64-runner"
      url = "https://gitlab.com"
      token = "YOUR_GITLAB_GLRT_TOKEN"
      executor = "kubernetes"
      environment = ["FF_USE_LEGACY_KUBERNETES_EXECUTION_STRATEGY=true"]

      [runners.kubernetes]
        namespace = "gitlab-runner"
        image = "bitnami/kubectl:latest"
        privileged = true
        helper_image = "registry.gitlab.com/gitlab-org/gitlab-runner/gitlab-runner-helper:arm64-v18.6.1"
        poll_timeout = 360
        
        [runners.kubernetes.volumes]
          [[runners.kubernetes.volumes.secret]]
            name = "k3d-kubeconfig"
            mount_path = "/kube"
            readOnly = true
            items = [{ key = "config", path = "config" }]
```

4. Install GitLab Runner via Helm
```
helm repo add gitlab https://charts.gitlab.io
helm repo update

helm install gitlab-runner gitlab/gitlab-runner \
  -n gitlab-runner \
  -f values.yaml
```

Check readiness:
```
kubectl get pods -n gitlab-runner -w
```

You should see:
```
gitlab-runner-xxxxx   1/1   Running
```
5. Grant Kubernetes RBAC Permissions

Allows CI jobs to use kubectl (demo!):
```
kubectl create clusterrolebinding gitlab-runner-job-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=gitlab-runner:default
```
6. Example .gitlab-ci.yml

Create:

.gitlab-ci.yml


Paste:
```
stages:
  - test

k3d-test:
  stage: test
  tags:
    - k3d-kubernetes-runner
  image: bitnami/kubectl:latest
  script:
    - echo "Hello from k3d!"
    - kubectl version --client
    - kubectl get nodes
```
Expected Pipeline Output
```
Hello from k3d!
Client Version: v1.34.2
NAME               STATUS   ROLES
k3d-dev-agent-0    Ready
k3d-dev-server-0   Ready
```

- Pipeline success
- GitLab Runner working in k3d
- ARM64 helper image loaded
- Full Kubernetes access from CI jobs

Architecture Overview
```
GitLab.com
    |
    v
GitLab Runner (in k3d)
    |
    v
Kubernetes Jobs (Pod per CI job)
```

Why this setup?

Perfect for DevOps Engineers building local CI/CD automation  
Ideal for learning Kubernetes, GitLab, Helm  
Great environment for CKA/CKAD exam practice  
No cloud cost  
Fully ARM64-native  

License
MIT — feel free to copy, modify, use and share.
