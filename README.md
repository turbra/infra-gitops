OpenShift GitOps Operator

Red Hat OpenShift GitOps is an Operator that uses Argo CD as the declarative GitOps engine. It enables GitOps workflows across multicluster OpenShift and Kubernetes infrastructure. Using Red Hat OpenShift GitOps, administrators can consistently configure and deploy Kubernetes-based infrastructure and applications across clusters and development lifecycles. Red Hat OpenShift GitOps is based on the open source project Argo CD and provides a similar set of features to what the upstream offers, with additional automation, integration into Red Hat OpenShift Container Platform and the benefits of Red Hat’s enterprise support, quality assurance and focus on enterprise security.

Installing OpenShift GitOps

The steps for deploying Red Hat OpenShift GitOps Operator are documented here.

Setup & configure the first GitOps repo

Create a deploy token in the infrastructure_gitops repository. Set the scope to read_repository.

Store the generated token credentials in your Secrets Manager.

From the bastion host, create a new secret in the openshift-gitops namespace using the token credentials from step 1:

```
oc create secret generic repo-infrastructure-gitops --from-literal=type=git --from-literal=url=https://<gitlab-repo>.git --from-literal=username=<token username> --from-literal=password=<token> -n openshift-gitops
```
Add a label to the secret to let ArgoCD know to use it to authenticate to the repository:
```
oc label secret repo-infrastructure-gitops argocd.argoproj.io/secret-type=repository -n openshift-gitops
```
Create an ApplicationSet that will deploy Applications to the cluster. This will create ArgoCD Applications from each directory under clusters/lab/live/* path of the infrastructure_gitops repository.

The example below is locked to a specific git commit (see revision & targetRevision) and should be adjusted accordingly (e.g. HEAD, v1.0.0, my-feature-branch, …).
```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-configs
  namespace: openshift-gitops
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - git:
      repoURL: https://<gitlab-repo>.git
      revision: d4108a0416748e564e71fbca73d3ebc3f0b6fac4
      directories:
      - path: clusters/live/lab/*
      - path: clusters/live/lab/base-configs
        exclude: true
  template:
    metadata:
      name: '{{.path.basename}}'
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: https://<gitlab-repo>.git
        targetRevision: d4108a0416748e564e71fbca73d3ebc3f0b6fac4
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'
      ignoreDifferences:
        - group: admissionregistration.k8s.io
          kind: MutatingWebhookConfiguration
          jsonPointers:
            - /webhooks/0/clientConfig/caBundle
      syncPolicy:
        automated:
          prune: true
          allowEmpty: true
```
Apply the ApplicationSet:
```
oc apply -f ./applicationset.yaml
```
Hashicorp Vault

Vault Helm Chart

A Helm Chart will be used to install and configure Vault. The chart is using the upstream Vault Helm Chart from Hashicorp and has been created in GitLab.

Installing

The ArgoCD ApplicationSet will try to apply the Helm chart in the cluster and no additional steps are needed for the installation.

A successful installation will deploy Hashicorp in High Availability mode:
```
$ oc get pod -n vault
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          13h
vault-1                                 1/1     Running   0          13h
vault-2                                 1/1     Running   0          13h
vault-agent-injector-59d67844c7-8664x   1/1     Running   0          13h
```
Initialization & Unseal

A new Vault installation will need to initialized and unsealed.

Start a remote shell in one of the Vault containers:
```
oc -n vault rsh vault-0
```
From the shell in the Vault container, initialize Vault:

vault operator init -key-shares=1 -key-threshold=1

If successful, the above will output the Unseal key and the initial root token. Save these to a safe location (for example AWS Secrets Manager).

Unseal Vault and use the Unseal key from the previous step to unseal Vault.
```
vault operator unseal
```
You can now exit the shell in the Vault container

Verify that Vault was successfully unsealed:
```
$ oc -n vault exec -it vault-0 -- vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.18.3
Build Date              2024-12-16T14:00:53Z
Storage Type            raft
Cluster Name            vault-cluster-05adb2c9
Cluster ID              45b63daa-5e08-7099-9ede-04eb4ca5aa3a
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2025-01-28T22:18:57.086450158Z
Raft Committed Index    39
Raft Applied Index      39
```
Repeat the unseal process on the remaining two pods (vault-1 & vault-2), using the Unseal key generated on the first pod. There’s no need repeat the init step on these pods. After all pods have been unsealed, the status should look similar to the below (notice HA Enabled, HA Cluster & HA Mode):
```
$ vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.18.3
Build Date              2024-12-16T14:00:53Z
Storage Type            raft
Cluster Name            vault-cluster-bcbe9de3
Cluster ID              74662814-22dd-51a5-3cd6-65ef8a9b2eaa
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 standby
Active Node Address     http://10.128.4.37:8200
Raft Committed Index    49
Raft Applied Index      49
```
The unseal step is required after a restart of Vault and requires the unseal key. Without this step, any data stored in Vault will be inaccessible.

Vault also supports auto unseal, see Vault documentation. 
