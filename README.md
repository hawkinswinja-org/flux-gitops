# 🚀 Flux Demo: GitOps with Flux

This demo introduces **Flux**, a GitOps toolset built for Kubernetes. It enables continuous delivery by syncing cluster state with definitions stored in Git.

*[Flux Docs Getting Started](https://fluxcd.io/flux/get-started/)*

## 🧩 1. Flux Components and `gotk`

Flux is made up of modular components, each responsible for a specific task. These include:

- `source-controller`: Manages sources like Git and Helm repositories.
- `kustomize-controller`: Applies Kubernetes manifests using Kustomize.
- `helm-controller`: Reconciles HelmReleases from Helm sources.
- `notification-controller`: Manages alerts and event forwarding.
- `image-reflector-controller`: Monitors image repositories for updates.
- `image-automation-controller`: Automates updates to Git based on image policies.

The `gotk` (GitOps Toolkit) CLI is used to interact with these components.

```bash
# Install gotk CLI (optional if using flux CLI only)
brew install fluxcd/tap/flux
```

## ⚙️ 2. Bootstrap Flux Cluster with Extra Components
Bootstrapping sets up Flux controllers in your cluster and connects it to a Git repository.
Set the environment variable GITHUB_TOKEN (Personal Access token), or fill it when prompted
```bash
flux bootstrap --help 
flux bootstrap github \
  --owner=hawkinswinja-org \
  --token-auth \ # use github token instead of ssh keys
  --repository=gitops \
  --branch=main \
  --path=clusters/dev \
  --personal \ # if using personal account
  --private=true \sets the repository to private. True by default
  --components-extra=image-reflector-controller,image-automation-controller
```
Installs the components in flux-system namespace.

## 🔧 3. flux create Supported Resources
Flux can manage several Kubernetes custom resources via flux create. Below are examples of some resources this demo will look into.

1. GitRepository
2. HelmRepository
3. Kustomization
4. HelmRelease
5. ImageRepository
6. ImagePolicy
7. ImageUpdateAutomation

Each of these can be created imperatively using Flux CLI or declaratively through manifest files.

## 🧬 4. flux create source git
A GitRepository source fetches manifests from a Git repo. Not concerned with what resources to sync to the cluster.
```bash
# create secret for authentication
flux create secret git podinfo-auth \
--url=ssh://git@github.com/hawkinswinja-org/podinfo \
--private-key-file=./.ssh/id_rsa --export 

flux create source git my-app \
  --url=https://github.com/my-org/my-app \
  --branch=main \
  --interval=1m \
  --secret-ref podinfo-auth \
  --export > podinfo-git-source.yaml
```
## 🪖 5. flux create source helm
Flux natively supports Helm charts from existinf Helm repos. The Helm chart can be a gitrepository or a published helm chart artifact. Simlar to how you run helm add repo command to get a helm repository
```bash
flux create source helm --help
flux create source helm bitnami \
  --url=https://charts.bitnami.com/bitnami \
  --interval=10m \
  --export > bitnami-source.yaml
```
To install a chart from the helm repo we use a helmrelease and reference the helmrepo as the provider. If using a Gitrepo, the folder path is the value for the --chart
```bash
flux create helmrelease --help
flux -n app create hr podinfo \
    --source=HelmRepository/podinfo \
    --chart=podinfo \
    --values-from=Secret/my-secret-values
    --export
```

## 🖼️ 6. Flux Image Automation (ImageRepository, ImagePolicy, ImageUpdate)
Image automation are extra components that monitor the image repository for the latest changes and depending on the image policy, update the source.
```bash
kubectl create secret docker-registry podinfo-docker-secret --docker-username=user --docker-password=password
# create an image repository to tell flux which registry to scan for new tags
flux create image repository podinfo \
--image=ghcr.io/stefanprodan/podinfo \
--interval=5m \
--secret-ref=podinfo-docker-secret \
--export
# Create an ImagePolicy to tell Flux which semver range to use when filtering tags
flux create image policy podinfo \
--image-ref=podinfo \
--select-semver=5.0.x \
--export

# update your deployment manifests to tell flux which policy to use for image update  # {"$imagepolicy": "flux-system:podinfo"}
# Create an ImageUpdateAutomation to tell Flux which Git repository to write image updates to
flux create image update flux-system \
--interval=30m \
--git-repo-ref=flux-system \
--git-repo-path="./clusters/my-cluster" \
--checkout-branch=main \
--push-branch=main \
--author-name=fluxcdbot \
--author-email=fluxcdbot@users.noreply.github.com \
--commit-template="{{range .Changed.Changes}}{{print .OldValue}} -> {{println .NewValue}}{{end}}" \
--export
```

## 📡 7. Flux Webhook Receivers
Flux supports webhook receivers to trigger immediate reconciliations from external systems such as GitHub.

Create webhook-token secret for authentication
```sh
# generate a random token for authentication to your cluster
TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
echo $TOKEN

# create a k8s native secret in the same namespace
kubectl -n flux-system create secret generic webhook-token \
--from-literal=token=$TOKEN

# create the webhook receiver
flux create receiver github-webhook \
  --type=github \
  --event=push \
  --secret-ref=webhook-token \
  --resource=GitRepository/my-app \
  --export > receiver.yaml
```

# 🚨 8. Flux Notifications (Alert Providers and Alerts)
Flux notification are used for alerting other supported third party tools to provide real time alerts to cluster admins.

Some supported tools include msteams, slack, pagerduty, email, sms.
The alert-provider authenticates to the third party tool, while the alert is responsible for monitoring changes in the listed eventsources and pushing the notification to the provider.

```bash
flux create alert-provider --help
flux create alert --help
```

## ToDos
1. GitOps in AKS advantages
2. Flux in multicluster (staging cluster and production cluster)
3. Promoting deployment to production
