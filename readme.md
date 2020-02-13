# Lemmings

Kubernetes cluster configuration that uses GitOps to keep state.

Includes Flux, Helm, cert-manager, Nginx Ingress Controller and Sealed Secrets.

* [fluxcd.io](https://fluxcd.io)
* [helm.sh](https://helm.sh)
* [github.com/jetstack/cert-manager](https://github.com/jetstack/cert-manager)
* [kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
* [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)

## Contributors

* Ivar Abrahamsen : [@flurdy](https://twitter.com/flurdy) : [github.com/flurdy](https://github.com/flurdy) : eray.uk


## Versions

* 2020-02-13 Flux 1.1, fluxcd.io annotations, and Helm 3
* 2019-11-07 Flux 0.16, flux.weave.works annotations and Helm 2


## Pre requisite

### Kubernetes tools and cluster

*     brew install kubectl
* Create Kubernetes cluster (see Kubernetes as a Service providers below)
* Set up Kubernetes context (see provider CLIs and `kubectx` below)
* Test cluster connection:

      kubectl cluster-info

## Lemmings install

### Fork/Clone repository

*
      brew install hub;
      hub clone flurdy/lemmings my-lemmings;
      cd my-lemmings;
      hub create -p my-lemmings;
      hub remote add upstream flurdy/lemmings;
      git push -u origin master

* Replace _my-lemmings_ with whatever you want to call your cluster
* Flux can also talk to Bitbucket, Gitlab and self-hosted repos

### Install Helm

* Helm released v3 in November 2019.
* If you have other clusters using Helm 2 tread carefully to make sure you can support both v2 and v3.


#### Helm 3

      brew install helm@3

* Vanilla Helm 3 includes no chart repositories. Lets add the core one

      helm repo add stable \
          https:// kubernetes-charts.storage.googleapis.com/

#### Helm 2

* If you need to use Helm 2, refer to an earlier version of Lemmings more aimed at Helm 2 + Tiller
* * https://github.com/flurdy/lemmings/tree/helm2
* Note you may need to specify specific _flux.weave.works_ annotations and HelmRelease versions to keep it working as things start to assume Helm 3

### Configure cert-manager

* Change email address in:
  * `issuer/issuer-staging.yml`
  * `issuer/issuer-prod.yml`
*     git add -p issuer;
      git commit -m "Updated email in certificate issuers";
      git push

### Install Flux

    helm repo add fluxcd https://charts.fluxcd.io;
    helm repo update;
    kubectl create namespace flux;
    kubectl apply -f flux/crd-flux-helm.yml;
    helm upgrade -i flux fluxcd/flux \
       --set git.url=git@github.com:YOURUSERNAME/my-lemmings \
       --namespace flux

* Replace _YOURUSERNAME/me-lemmings_ with your username and repository.

#### Install Flux Helm Operator

    helm upgrade -i helm-operator fluxcd/helm-operator \
      --set git.ssh.secretName=flux-git-deploy \
      --namespace flux \
      --set helm.versions=v3

####  Fluxctl & SSH

* Install `fluxctl` and generate SSH key

      brew install fluxctl;
      export FLUX_FORWARD_NAMESPACE=flux;
      fluxctl identity --k8s-fwd-ns flux

* Add Flux's SSH key to your github repo with write access:
   * [github.com/YOURUSERNAME/lemmings/settings/keys](https://github.com/YOURUSERNAME/lemmings/settings/keys)

* Tail log:

      kubectl logs -n flux deploy/flux -f

## Your GitOps based Kubernetes cluster is live!

If you waited a few minutes then your GitOps configured Kubernetes cluster is now live.

## Your first application

* (We baked one earlier for you: `workflows/hello`)

* View/Edit *workflows/hello/deployment.yml*

      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello-deployment
        annotations:
          fluxcd.io/automated: "true"
      spec:
        selector:
          matchLabels:
            app: hello
        replicas: 2
        template:
          metadata:
            labels:
              app: hello
          spec:
            containers:
            - name: hello-container
              image: nginxdemos/hello:0.2
              ports:
              - containerPort: 80

    * Note the `fluxcd.io/automated` annotation which tells Flux to upgrade it when possible.

* Edit *workflows/hello/service.yml*

      apiVersion: v1
      kind: Service
      metadata:
        name: hello-service
      spec:
        selector:
           app: hello
        ports:
        - protocol: TCP
           port: 80
           targetPort: 80

* Edit *workflows/hello/ingress.yml*

      apiVersion: extensions/v1beta1
      kind: Ingress
      metadata:
        name: hello-ingress
        annotations:
          kubernetes.io/ingress.class: nginx
      #    cert-manager.io/cluster-issuer: letsencrypt-staging
      spec:
      #  tls:
      #  - hosts:
      #    - hello.example.com
      #    secretName: letsencrypt-certificate-staging
        rules:
        - host: hello.example.com
          http:
            paths:
            - backend:
                serviceName: hello-service
                servicePort: 80

   The SSL certificate part is commented out for now as it wont work without a real domain that *Letsencrypt* can do a reverse verification call to.

### Launch application

* If you made any changes:

      git add app/hello
      git commit -m "App: Hello"
      git push

* Wait (max 5 minutes) for *Flux* to detect your changes and apply them
* Or if impatient:

      fluxctl sync
* Check if the Hello cluster resources are running

      kubectl get deployment hello-deployment
      kubectl get service hello-service
      kubectl get ingress hello-ingress
      kubectl get pods -l app=hello


### Test application

* Find the ingress controller's `External IP`

      kubectl get services nginx-ingress-controller

* Use `curl` to resolve the URL. Replace `11.22.33.44` with the external IP.
* And `lynx` to view it

      curl -H "Host: hello.example.com" \
        --resolve hello.example.com:80:11.22.33.44 \
        --resolve hello.example.com:443:11.22.33.44 \
        http://hello.example.com | lynx -stdin
* This should show a basic hello world page, with an Nginx logo and some server address, name and date details.

### Update application

* Now if ever [hub.docker.com/r/nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello) bumps its version to higher than *0.2*,
   Flux will detect it and bump your version in Kubernetes, and also commit the change back to your Git repository.
* Unfortunately that image has not been updated for 2 years and unlikely to be updated, but when you start to use your own images they will be automagically updated if the deployment's annotation `fluxcd.io/automated` is set to true.

### Delete application

    git rm -r apps/hello
    git commit -m "Removed app Hello"
    git push
    fluxctl sync
    kube delete ingress hello-ingress
    kube delete service hello-service
    kube delete deployment hello-deployment

## Sealed Secrets

* If you need [secrets](https://kubernetes.io/docs/concepts/configuration/secret/),
(for private docker registry authentication, database passwords, API tokens),
DO NOT commit them in plain text to the Git repository.
* Instead we will use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) to encrypt them.
* Check if Sealed Secrets was installed by Flux:

      kube get services sealed-secrets -n kube-system

* Install CLI

      brew install kubeseal

* Fetch the cluster's Sealed Secrets' public key

      kubeseal --fetch-cert \
         --controller-namespace=kube-system \
         --controller-name=sealed-secrets \
         > sealed-secrets/sealed-secrets-cert.pem
      git add sealed-secrets/sealed-secrets-cert.pem
      git commit -m "Sealed secret public key"

* Create secrets but do not apply them to the cluster.

   * E.g with the `--dry-run` argument.

         mkdir secrets;
         kubectl create secret generic basic-auth \
            --from-literal=user=admin \
            --from-literal=password=admin \
            --dry-run \
            -o json > secrets/basic-auth.json

* Encrypt secret and transform to a sealed secret:

      kubeseal --format=yaml \
         --cert=sealed-secrets/sealed-secrets-cert.pem \
         < secrets/basic-auth.json \
         > secrets/secret-basic-auth.yml
      git add secrets/secret-basic-auth.yml
      rm secrets/basic-auth.json
      git commit -m "Sealed basic auth secret"
      git push

* After a little time you should see the secret in your cluster

      kubectl describe secret basic-auth

* Afterwards if you no longer need it, delete the file and secret:

      git rm secrets/secret-basic-auth.yml
      git commit -m "Removed basic auth"
      git push
      fluxctl sync
      kubectl delete secret basic-auth
      kubectl delete SealedSecret basic-auth

* Note: Step by step Docker Registry Cookbook
* * [flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html)

## Go wild

* Add/update your deployments, services, charts, docker registries, secrets, etc

## Don't touch

* Once Flux is running, by convention avoid using `kubectl create|apply` etc.
* Nearly all changes should be via Git and Flux.
* Any `kubectl` interaction should be read only.
* Flux will not remove some resources, so you might need `kubectl delete` occasionally.

## More information, alternatives, suggestions

* Kubernetes as a Service
  * Amazon AWS EKS: [aws.amazon.com/eks/](https://aws.amazon.com/eks/)
  * Google Cloud GKE: [cloud.google.com/kubernetes-engine/](https://cloud.google.com/kubernetes-engine/)
  * Microsoft Azure AKS: [azure.microsoft.com/en-us/services/kubernetes-service/](https://azure.microsoft.com/en-us/services/kubernetes-service/)
  * DigitalOcean Kubernetes: [www.digitalocean.com/products/kubernetes/](https://www.digitalocean.com/products/kubernetes/)

* Cloud provider CLI
  * [cloud.google.com/sdk/](https://cloud.google.com/sdk/)

        brew cask install google-cloud-sdk

  * [github.com/digitalocean/doctl](https://github.com/digitalocean/doctl)

        brew install doctl
  * [eksctl.io](https://eksctl.io/)

        brew tap weaveworks/tap
        brew install weaveworks/tap/eksctl

  * [github.com/Azure/azure-cli](https://github.com/Azure/azure-cli)

        brew install azure-cli

* [hub.github.com](https://hub.github.com)
* [docs.fluxcd.io/en/stable/references/fluxctl.html](https://docs.fluxcd.io/en/stable/references/fluxctl.html)
* [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx)

      brew install kubectx

* [github.com/vmware-tanzu/octant](https://github.com/vmware-tanzu/octant)

      brew install octant

* [k9ss.io](https://k9ss.io)

      brew install derailed/k9s/k9s

* [keel.sh](https://keel.sh)
* [lynx.browser.org](https://lynx.browser.org)
* [weave.works/blog/managing-helm-releases-the-gitops-way](https://www.weave.works/blog/managing-helm-releases-the-gitops-way)
* [github.com/fluxcd/helm-operator-get-started](https://github.com/fluxcd/helm-operator-get-started)
* [ramitsurana.github.io/awesome-kubernetes/](https://ramitsurana.github.io/awesome-kubernetes/)
* [github.com/fluxcd/multi-tenancy](https://github.com/fluxcd/multi-tenancy)
* [github.com/justinbarrick/fluxcloud](https://github.com/justinbarrick/fluxcloud)
* [github.com/flux-web/flux-web](https://github.com/flux-web/flux-web)
* [flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html](https://flurdy.com/docs/kubernetes/registry/kubernetes-docker-registry.html)

### Notes:

* Certain operation takes a few minutes, e.g. pod creation.
* cert-manager 0.11+ requires kubernetes 1.15+
* Client tools are also available on Linux, Windows and more.
