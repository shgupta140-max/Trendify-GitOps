# Trendify GitOps

Kubernetes deployment configuration for **TrendStore**, the Trendify web application. This repository is the deployment layer in the Trendify delivery workflow: application artifacts are built elsewhere, AWS infrastructure is provisioned separately, and Jenkins applies the versioned Kubernetes manifests to Amazon EKS.


## 🌐 Trendify Project Ecosystem

This repository is part of the **Trendify Enterprise Cloud Platform**, a fully automated, GitOps-driven, two-tier application stack. To enforce a strict separation of concerns, the architecture is decoupled into four distinct repositories:

1. **[Trendify-Platform](https://github.com/shgupta140-max/trendify-platform.git) (Automation & Observability):**
   * **Role:** The foundational layer. Contains Terraform code to provision the Jenkins CI/CD automation server and Helm configurations to deploy the centralized monitoring stack (`kube-prometheus-stack` & `blackbox-exporter`).

2. **[Trendify-Infra]( https://github.com/shgupta140-max/Trendify-Infra.git) (Cloud Infrastructure):**
   * **Role:** The immutable AWS infrastructure layer. Contains Terraform modules to provision the production-grade Amazon EKS cluster (`trendstore-cluster` in `ap-south-1`), VPC networks, IAM Access Entries, and the AWS ALB Controller.

3. **[Trendify-App](https://github.com/shgupta140-max/Trendify-App.git) (Application Code & CI):**
   * **Role:** The product layer. Houses the Node.js application source code, Dockerfile, and the Continuous Integration (CI) Jenkins pipeline. 
   * **Connection:** This pipeline builds the image, pushes it to DockerHub, and automatically commits the new image tag directly into the `Trendify-GitOps` repository.

4. **[Trendify-GitOps](https://github.com/shgupta140-max/Trendify-GitOps.git) (Cluster State & CD):**
   * **Role:** The single source of truth for the Kubernetes cluster state. Contains the application deployment manifests and Kustomize overlays.
   * **Connection:** Triggered by commits from `Trendify-App`, this Jenkins pipeline requires manual Slack approval before deploying changes to the `Trendify-Infra` EKS cluster and dynamically injecting the AWS ALB URL into the monitoring probes.

---

## Repository Responsibilities

This repository contains:

- The `trendstore` Kubernetes namespace.
- A two-replica `trendstore-deployment` running the container on port `3000`.
- A `ClusterIP` service named `trendstore-service`.
- An AWS Application Load Balancer ingress for `trendstore.nikboss.xyz`.
- A Jenkins pipeline that retrieves EKS credentials, requests deployment approval in Slack, applies Kustomize, and checks rollout status.

Application source code and AWS infrastructure are intentionally maintained in separate repositories.


## Deployment Architecture

```text
Trendify-App
	|
	v
Docker image: docker.io/shgupta140/trendify-app:<tag>
	|
	v
Trendify-GitOps -- Jenkins approval --> Amazon EKS: trendstore-cluster
	|
	v
ALB --> trendstore-service --> trendstore pods
```

## Current Deployment Configuration

| Setting | Value |
| --- | --- |
| AWS region | `ap-south-1` |
| EKS cluster | `trendstore-cluster` |
| Kubernetes namespace | `trendstore` |
| Deployment | `trendstore-deployment` |
| Replicas | `2` |
| Container port | `3000` |
| Image | `docker.io/shgupta140/trendify-app:v3` |
| Public hostname | `trendstore.nikboss.xyz` |
| Ingress | AWS ALB, internet-facing, HTTP redirected to HTTPS |

## Repository Layout

```text
.
├── Jenkinsfile
├── README.md
└── kubedefs/
    ├── 00-namespace.yml
    ├── 01-deployment.yml
    ├── 02-aws-alb-ingress.yml
    └── kustomization.yml
```

Kustomize applies the manifests from `kubedefs/` as a single deployment unit. The deployment currently declares the image directly in `01-deployment.yml`; keep that image reference and the image entry in `kustomization.yml` aligned when publishing a new version.

## Prerequisites

- AWS CLI authenticated to an account that can access the EKS cluster.
- `kubectl` and Kustomize support (`kubectl apply -k`).
- Access to the `trendstore-cluster` cluster in `ap-south-1`.
- Jenkins with the Slack notification integration configured.
- An AWS Load Balancer Controller installed in the EKS cluster.
- An ACM certificate in `ap-south-1` matching the hostname and referenced certificate ARN.
- DNS for `trendstore.nikboss.xyz` pointing to the ALB created by the ingress.

## Deploy Manually

```bash
aws eks update-kubeconfig --name trendstore-cluster --region ap-south-1
kubectl apply -k ./kubedefs
kubectl rollout status deployment/trendstore-deployment -n trendstore
kubectl get pods -n trendstore -o wide
kubectl get ingress trendstore-alb -n trendstore
```

To preview the rendered resources without applying them:

```bash
kubectl kustomize ./kubedefs
```

## Jenkins Pipeline

The `Jenkinsfile` performs these stages:

1. Updates the local kubeconfig for the EKS cluster.
2. Checks the available `kubectl` version.
3. Sends a Slack approval request and waits for an approval or abort.
4. Applies the manifests with `kubectl apply -k ./kubedefs`.
5. Waits for the deployment rollout and lists the running pods.
6. Sends success, failure, or aborted notifications to Slack.

The Jenkins agent must provide `aws`, `kubectl`, and the required AWS and Slack credentials. The pipeline also requires permission to call `slackSend` and pause for the approval input step.

## Change and Release Workflow

1. Build and publish a new application image from [Trendify-App](https://github.com/shgupta140-max/Trendify-App).
2. Update the image tag in `kubedefs/01-deployment.yml` and keep the image entry in `kubedefs/kustomization.yml` aligned.
3. Review the rendered output with `kubectl kustomize ./kubedefs`.
4. Commit and push the manifest change to this repository.
5. Run the Jenkins job and approve the deployment in Slack.
6. Verify rollout status, pods, ingress, and the public hostname.

## Operational Notes

- The ingress is internet-facing and uses the AWS Load Balancer Controller.
- The configured health check targets `/` on service port `3000`.
- The deployment currently uses a pinned image tag (`v3`). Avoid using `latest` for production releases so that every deployment is reproducible.
- The certificate ARN, ALB name, hostname, cluster name, and image tag are environment-specific values. Review them before reusing these manifests in another account or region.

## License

No license file is currently included in this repository.