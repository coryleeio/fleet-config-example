### What is this?

This repo is meant to serve as a jumping off point for a self managing argoCD install which adheres to gitops paradigms and manages a multi region set of k8s clusters

### What are the files/directories for?

###### config-root/ 
    
    This is generated manifests which are applied to the cluster, the generators are meant to be declarative and idempotent. 
    This directory can be deleted and regenerated from the helmfiles / kustomize templates
    This is what is actually applied to each cluster. You shouldn't be making any manual changes here, make changes elsewhere and regenerate this and commit it to apply it to the clusters.

    Each folder in here corresponds a k8s cluster
    though we use helmfile environments to represent them these dont have to be your environments. 

###### helmfiles/

    This is a collection of helmfiles which are installed to the cluster, each can be rendered by navigating to their directory and rendering them like this:
        
        'helmfile template --environment {cluster_name}'

    this is where you typically make changes to your app, after you do this you'll regenerate the manifests in config-root and commit them to apply it to the clusters.

###### kustomizations/

    This is a collection of kustomized manifests which are installed to the cluster, each can be rendered by navigating to their directory and rendering them like this:
        
        'kustomize build kustomizations/{app_name}/overlays/{cluster_name}'

    This is useful when you need to apply bare manifests to the clusters and make small changes to them for each environment.

    This is nice for keeping CRDs separate from helm charts as well so that removal of the chart doesn't remove the CRDs or when the chart doesn't ship CRDs

###### environments.yaml

    Mapping of which files are loaded by helmfile itself, they're merged in the order specified here then applied to the helmfile template, checkout helmfile docs for more information.

    This determines what k8s clusters we know exist and what order we traverse files to figure out if we should build a CRD, so it has to be kept up to date

    this is NOT what is actually passed to the charts! and so it is more or less duplicated in the values section of each chart
    https://blog.derlin.ch/helmfile-a-simple-trick-to-handle-values-intuitively#my-tip-on-using-values-in-helmfile

### How to bootstrap a cluster:

    assumed installs: docker(localdev), kind(localdev), step-cli (&openssl), argocd cli, kubectl, vault cli

    // Create some namespaces, by hand, argo will idempotently ensure these are created and manage them once the bootstrap is completed.
    kind create cluster

    kubectl create namespace vault
    kubectl create namespace argocd
    kubectl apply -f config-root/localdev/argocd/
    kubectl apply -f config-root/localdev/application-root/

    export GITHUB_TOKEN=yyyyyyyyyyyyyyyyyyyyyyyyyy
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    export ARGOCD_PASS=uuuuuuuuuuuuuuuuuu

    kubectl port-forward -n argocd svc/argocd-server 8000:80

    argocd login localhost:8000 --username admin --password $ARGOCD_PASS
    argocd repocreds add https://github.com/coryleeio --username coryleeio --password $GITHUB_TOKEN
    login to webui with admin/$(ARGOCD_PASS) at localhost:8000

    sync argo and vault

    kubectl port-forward -n vault svc/vault 8200:8200

    now argocd will apply everything in configuration root and for all of your clusters

### Add an application

    * write new manifests into helmfiles/ or kustomizations/
    * render the manifests to config-root
    * add an application crd to config-root/application-root
    * commit

### Remove an application

    * delete it's folder
    * remove it's manifests from config-root or rerender all manifests in config-root
    * remove application crd to config-root/application-root
    * commit

### Add/remove a cluster

    * update environments.yaml
    * add/remove new values to all of your helmcharts and add/remove any kustomization overlays
    * render the manifests to config-root
    * commit

### Updating an application version

    * Update the SHA of the image
    // targeted refreshes can be used when not add/removing applications
    // so each app doesn't have to regenerate all the manifests
    * render the manifests to config-root
    * commit
