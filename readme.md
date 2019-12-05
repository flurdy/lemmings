# Lemmings

Kubernetes cluster configuration that uses GitOps to keep state.

Includes Flux, Helm, cert-manager, Nginx Ingress Controller and Sealed Secrets.

* [fluxcd.io](https://fluxcd.io)
* [helm.sh](https://helm.sh)
* [github.com/jetstack/cert-manager](https://github.com/jetstack/cert-manager)
* [kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
* [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets)

## Author

* [@flurdy](https://twitter.com/flurdy) : [github.com/flurdy](https://github.com/flurdy)

### Disclaimer

On 2019-11-07, this all worked with the current versions of tools.
In time some may need tweaking, version bumping, etc.



## Pre requisite

### Kubernetes tools and cluster

*     brew install kubectl
* Create Kubernetes cluster (see Kubernetes as a Service providers below)
* Set up Kubernetes context (see provider CLIs and `kubectx` below)
* Test cluster connection:

      kubectl cluster-info

### Fork/Clone repository

    brew install hub
    hub clone flurdy/lemmings
    cd lemmings
    hub fork --remote-name=origin

### Install Helm

This is for Helm v2. The brand new v3 does not require Tiller.

    brew install kubernetes-helm@2.16.1
    kubectl create -f tiller/serviceaccount-tiller.yml
    kubectl create -f tiller/rbac-tiller.yml
    helm init --service-account tiller --history-max 200

### Configure cert-manager

* Change email address in:
  * `issuer/issuer-staging.yml`
  * `issuer/issuer-prod.yml`
*     git add -p issuer
      git commit -m "Updated email address in certificate issuers"
      git push

### Install Flux

    helm repo add fluxcd https://charts.fluxcd.io
    kubectl apply -f flux/crd-flux-helm.yml
    helm upgrade -i flux \
       --set helmOperator.create=true \
       --set helmOperator.createCRD=false \
       --set git.url=git@github.com:YOURUSERNAME/lemmings \
       --namespace flux \
       fluxcd/flux


* Install `fluxctl` and generate SSH key


      brew install fluxctl
      export FLUX_FORWARD_NAMESPACE=flux
      fluxctl identity --k8s-fwd-ns flux

* Add Flux's SSH key to your github repo with write access:
   * https://github.com/YOURUSERNAME/lemmings/settings/keys


## Your GitOps K8s is live!

Your GitOps configured Kubernetes cluser is now live.

## Your first application

* We baked one earlier for you `apps/hello`:

* View/Edit *apps/hello/deployment.yml*

      apiVersion: apps/v1
      kind: Deployment
      metadata:
         name: hello-deployment
      annotations:
         flux.weave.works/automated: "true"
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

    * Note the `flux.weave.works/automated` annotation.
       * It will change to `fluxcd.io/automated` in Future flux versions.

* Edit *apps/hello/service.yml*

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

* Edit *apps/hello/ingress.yml*

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

* Tail log:

      kubectl logs -n flux deploy/flux -f

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

      curl -H "Host: hello.example.com" \
        --resolve hello.example.com:80:11.22.33.44 \
        --resolve hello.example.com:443:11.22.33.44 \
        http://hello.example.com


### Update application

* Now if ever [hub.docker.com/r/nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello) bumps its version to higher than *0.2*, Flux will detect it and bump your version in Kubernetes, and also commit the change to your Git repository.
* Unfortunately that image has not been updated for 2 years and unlikely to be updated, but when you start to use your own images they will be automagically updated if the deployment's annotation `flux.weave.works/automated` is set to true.

## Sealed Secrets

* If you need [secrets](https://kubernetes.io/docs/concepts/configuration/secret/),
(for passwords, keys, tokens, etc),
do not commit them in plain text to the Git repository.
* Instead we will use [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) to encrypt them.
* Check if Sealed Secrets was installed by Flux:

      kube get services sealed-secrets -n kube-system

* Install CLI

      brew install kubeseal

* Fetch the cluster's Sealed Secrets' public key

      kubeseal --fetch-cert \
         --controller-namespace=kube-system \
         --controller-name=sealed-secrets \
         > secrets/sealed-secrets-cert.pem

* You can add this public key to git if you want.

      git add secrets/sealed-secrets-cert.pem
      git commit -m "Sealed secret public key"

* Create secrets but do not apply them to the cluster.

   * E.g with the `--dry-run` argument.

         kubectl create secret generic basic-auth \
            --from-literal=user=admin \
            --from-literal=password=admin \
            --dry-run \
            -o json > secrets/basic-auth.json

* Encrypt secret and transform to a sealed secret:

      kubeseal --format=yaml \
         --cert=secrets/sealed-secrets-cert.pem \
         < secrets/basic-auth.json \
         > secrets/secret-basic-auth.yml
      rm secrets/basic-auth.json
      git add secrets/secret-basic-auth.yml
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

        brew install google-cloud-sdk

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
* [weave.works/blog/managing-helm-releases-the-gitops-way](https://www.weave.works/blog/managing-helm-releases-the-gitops-way)
* [github.com/fluxcd/helm-operator-get-started](https://github.com/fluxcd/helm-operator-get-started)
* [ramitsurana.github.io/awesome-kubernetes/](https://ramitsurana.github.io/awesome-kubernetes/)
* [github.com/fluxcd/multi-tenancy](https://github.com/fluxcd/multi-tenancy)

### Notes:

* Certain operation takes a few minutes, e.g. pod creation.
* cert-manager 0.11+ requires kubernetes 1.15+
* Client tools are also available on Linux, Windows and more.
