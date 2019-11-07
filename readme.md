# Lemmings

Kubernetes cluster configuration that uses GitOps to keep state.

Use Flux, Helm, cert-manager.

* [fluxcd.io](https://fluxcd.io)
* [helm.sh](https://helm.sh)
* [github.com/jetstack/cert-manager](https://github.com/jetstack/cert-manager)

## Author

* [@flurdy](https://twitter.com/flurdy) : [github.com/flurdy](https://github.com/flurdy)

## Pre requisite

### Kubernetes tools and cluster

* `brew install kubectl`
* Create Kubernetes cluster
* Set up Kubernetes context
* Test cluster connection:

   `kubectl cluster-info`

### Fork/Clone repository

* `brew install hub`
* `hub clone flurdy/lemmings`

### Helm

* `brew install kubernetes-helm`
* `kubectl create -f tiller/serviceaccount-tiller.yml`
* `kubectl create -f tiller/rbac-tiller.yml`
* `helm init --service-account tiller --history-max 200`

### Configure cert-manager

* Change email address in:
  * `issuer/issuer-staging.yml`
  * `issuer/issuer-prod.yml`

### Flux

* `helm repo add fluxcd https://charts.fluxcd.io`
* `kubectl apply -f flux/crd-flux-helm.yml`
*
       helm upgrade -i flux \
       --set helmOperator.create=true \
       --set helmOperator.createCRD=false \
       --set git.url=git@github.com:YOURUSERNAME/lemmings \
       --namespace flux \
       fluxcd/flux
* `brew install fluxctl`
* `export FLUX_FORWARD_NAMESPACE=flux`
* Add flux ssh key to github repo with write access:
  * `fluxctl identity --k8s-fwd-ns flux`
  * github.com/YOURUSERNAME/lemmings/settings/keys

## Run wild

* Add/update your deployments, services, charts, etc
* Commit and push
* Tail log: `kubectl logs -n flux deploy/flux -f`
* Wait (max 5 minutes) for *Flux* to detect change and apply them
* Or `fluxctl sync`

## More information, alternatives, suggestions

* Kubernetes as a Service
  * Amazon AWS EKS: [aws.amazon.com/eks/](https://aws.amazon.com/eks/)
  * Google Cloud GKE: [cloud.google.com/kubernetes-engine/](https://cloud.google.com/kubernetes-engine/)
  * Microsoft Azure AKS: [azure.microsoft.com/en-us/services/kubernetes-service/](https://azure.microsoft.com/en-us/services/kubernetes-service/)
  * DigitalOcean Kubernetes: [www.digitalocean.com/products/kubernetes/](https://www.digitalocean.com/products/kubernetes/)

* Cloud provider CLI
  * [cloud.google.com/sdk/](https://cloud.google.com/sdk/)
    * `brew install google-cloud-sdk`
  * [github.com/digitalocean/doctl](https://github.com/digitalocean/doctl)
    * `brew install doctl`
  * [eksctl.io](https://eksctl.io/)
    * `brew tap weaveworks/tap`

      `brew install weaveworks/tap/eksctl`
  * [github.com/Azure/azure-cli](https://github.com/Azure/azure-cli)
    * `brew install azure-cli`
* [hub.github.com](https://hub.github.com)
* [docs.fluxcd.io/en/stable/references/fluxctl.html](https://docs.fluxcd.io/en/stable/references/fluxctl.html)
* [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx)
  * `brew install kubectx`
* [github.com/vmware-tanzu/octant](https://github.com/vmware-tanzu/octant)
  * `brew install octant`

### Notes:

* Certain operation takes a few minutes, e.g. pod creation.
* cert-manager 0.11+ requires kubernetes 1.15+
* Client tools are also available on Linux, Windows and more.
