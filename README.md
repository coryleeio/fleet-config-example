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

    assumed installs: docker(localdev), kind(localdev), step-cli (&openssl), argocd cli, kubectl, linkderd cli, vault cli, step-cli

    // Create some namespaces, by hand, argo will idempotently ensure these are created and manage them once the bootstrap is completed.
    kind create cluster --config=kind-config.yaml

    step-cli certificate create root.linkerd.cluster.local secret/ca.crt secret/ca.key \
    --profile root-ca --no-password --insecure

    kubectl apply -k kustomizations/namespaces/overlays/localdev

    kubectl create secret tls \
    linkerd-trust-anchor \
    --cert=secret/ca.crt \
    --key=secret/ca.key \
    --namespace=linkerd

    copy secret/ca.crt into helmvalues/linkerd-control-plane/values.yaml if its not already, careful of indentation

    kubectl apply -n localdev-argocd -k kustomizations/argocd/overlays/localdev
    kubectl apply -k kustomizations/application-root/overlays/localdev

    argocd account bcrypt --password admin
    # example is set to 'admin'
    kubectl -n localdev-argocd patch secret argocd-secret \
    -p '{"stringData": {
        "admin.password": "$2a$10$ZgpJGNnrnxwbTToFHQ6A/ucl5ddGGzpT.v4bOAmPrTRqhMoJsgdx2",
        "admin.passwordMtime": "'$(date +%FT%T%Z)'"
    }}'

    export GITHUB_TOKEN=yyyyyyyyyyyyyyyyyyyyyyyyyy
    kubectl -n localdev-argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    export ARGOCD_PASS=admin

    kubectl port-forward -n localdev-argocd svc/argocd-server 8000:80

    argocd login localhost:8000 --username admin --password $ARGOCD_PASS
    argocd repocreds add https://github.com/coryleeio --username coryleeio --password $GITHUB_TOKEN

    kubectl delete pods -n localdev-argocd --all



    login to webui with admin/$(ARGOCD_PASS) at localhost:8000

    sync argocd, application-root, and vault
    
    kubectl port-forward -n localdev-vault svc/localdev-vault 8200:8200
    kubectl port-forward -n linkerd-viz svc/web 8084:8084
    kubectl port-forward -n localdev-prometheus svc/localdev-prometheus-server 9000:80
    kubectl port-forward -n linkerd-viz svc/prometheus 9001:9090
    kubectl port-forward -n localdev-grafana svc/localdev-grafana 10000:80
    kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097

    export VAULT_ADDR=http://localhost:8200
    vault login root
    vault auth enable kubernetes
    vault secrets disable secret
    vault secrets enable --path=apps -version=2 kv
    vault secrets enable --path=administrator -version=2 kv

    vault policy write vault-administrator vault-iam/vault-administrator.hcl
    vault policy write administrator-read-write vault-iam/administrator-read-write.hcl
    vault policy write administrator-read-only vault-iam/administrator-read-only.hcl
    vault policy write apps-read-write vault-iam/apps-read-write.hcl
    vault policy write apps-read-only vault-iam/apps-read-only.hcl

    vault write auth/kubernetes/role/linkerd-vault-secret-sync \
    bound_service_account_names=linkerd-vault-secret-sync \
    bound_service_account_namespaces=linkerd \
    policies=administrator-read-only \
    ttl=1h


    # Restart everything but the k8s control plane once everything is green, then resync as needed to ensure everything is meshed

    kubectl -n linkerd-viz delete pods --all
    kubectl -n localdev-argocd delete pods --all
    kubectl -n localdev-cert-manager delete pods --all
    kubectl -n localdev-external-secrets delete pods --all
    kubectl -n localdev-grafana delete pods --all
    kubectl -n localdev-internal-ingress-nginx delete pods --all
    kubectl -n localdev-loki delete pods --all
    kubectl -n localdev-prometheus delete pods --all
    kubectl -n localdev-promtail delete pods --all
    kubectl -n localdev-tempo delete pods --all
    kubectl -n localdev-vault delete pods --all


    # get kind control plane ip
    kubectl get node kind-control-plane -o=jsonpath='{range .status.addresses[*]}{.type}{"\t"}{.address}{"\n"}'

    # use it to access kafka via node port in a way kafka will be happy with
    kafka-topics.sh --bootstrap-server 172.18.0.2:31111 --topic first_topic --create --partitions 3 --replication-factor 1

### Add/Remove an application

    * add/remove a new application crd to kustomizations/application-root, update kustomization.yaml as needed.
    * if helm add/remove values files to helmvalues/
    * commit
