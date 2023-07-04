### What is this?

This repo is meant to serve as a jumping off point for a self managing argoCD install which adheres to gitops paradigms. It does any initial bootstrapping for a set of k8s clusters which may span multiple regions in multiple environments. 

### What are the files/directories for?

###### helmvalues/

    This is a collection of helm values files which are fed into helm charts defined by argocd in the application root

###### kustomizations/

    This is a collection of kustomized manifests which are installed to the cluster, each can be rendered by navigating to their directory and rendering them like this:
        
        'kustomize build kustomizations/{app_name}/overlays/{cluster_name}'

    This is useful when you need to apply bare manifests to the clusters and make small changes to them for each environment.

### How to bootstrap a cluster:

    assumed installs: docker(localdev), kind(localdev), step-cli (&openssl), argocd cli, kubectl, linkderd cli, vault cli

    // Create some namespaces, by hand, argo will idempotently ensure these are created and manage them once the bootstrap is completed.
    kind create cluster

    kubectl apply -k kustomizations/namespaces/overlays/localdev
    kubectl apply -n argocd -k kustomizations/argocd/overlays/localdev
    kubectl apply -f config-root/localdev/application-root/

    export GITHUB_TOKEN=yyyyyyyyyyyyyyyyyyyyyyyyyy
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    export ARGOCD_PASS=uuuuuuuuuuuuuuuuuu

    kubectl port-forward -n argocd svc/argocd-server 8000:80

    argocd login localhost:8000 --username admin --password $ARGOCD_PASS
    argocd repocreds add https://github.com/coryleeio --username coryleeio --password $GITHUB_TOKEN
    login to webui with admin/$(ARGOCD_PASS) at localhost:8000

    sync argocd, application-root, and vault
    kubectl port-forward -n vault svc/localdev-vault 8200:8200

### Add/Remove an application

    * add/remove a new application crd to kustomizations/application-root, update kustomization.yaml as needed.
    * if helm add/remove values files to helmvalues/
    * commit
