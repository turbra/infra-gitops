This repository contains the configurations I utilize for my OpenShift clusters, taking cues from Gerald Nunn's [Standards](https://github.com/gnunn-gitops/standards).

For more insights into OpenShift GitOps and Kustomize, check out Mark Roberts blog post [Your Guide to Continuous Delivery with OpenShift GitOps and Kustomize](https://cloud.redhat.com/blog/your-guide-to-continuous-delivery-with-openshift-gitops-and-kustomize).

# Install OpenShift GitOps

```
oc apply -k install-gitops
```
## Workflow summary
  
  When you run the command `oc apply -k install-gitops`, the following happens:

  1. `oc` (OpenShift CLI) recognizes the `-k` flag and invokes Kustomize on the `install-gitops/` directory.
  2. Kustomize reads the `kustomization.yaml` file to determine which resources to include and any modifications to make.
  3. The resources defined are processed and prepared for deployment to the cluster.
  4. The processed resources are applied to the OpenShift cluster. This results in:
     - The necessary permissions being granted via the ClusterRoleBinding.
     - The GitOps Operator being installed or updated based on the Subscription.
---

# Setup & configure the first GitOps repo

If your repository is set to public you can skip the steps for creating deploy token(s) and secrets.

Create a deploy token in the infrastructure_gitops repository. Set the scope to `read_repository`.

Store the generated token credentials in your Secrets Manager.

Create a new secret in the openshift-gitops namespace using the deploy token credentials you just created:

```
oc create secret generic repo-infrastructure-gitops --from-literal=type=git --from-literal=url=https://<gitlab-repo>.git --from-literal=username=<token username> --from-literal=password=<token> -n openshift-gitops
```
Add a label to the secret to let ArgoCD know to use it to authenticate to the repository:
```
oc label secret repo-infrastructure-gitops argocd.argoproj.io/secret-type=repository -n openshift-gitops
```
Create an ApplicationSet that will deploy Applications to the cluster. This will create ArgoCD Applications from each directory under clusters/lab/live/* path of the infrastructure_gitops repository.

The example below is locked to a specific branch (see revision & targetRevision) and should be adjusted accordingly (e.g. HEAD, v1.0.0, my-feature-branch, …).
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
      repoURL: https://github.com/turbra/infra-gitops.git
      revision: main
      directories:
      - path: clusters/lab/live/*
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
        repoURL: https://github.com/turbra/infra-gitops.git
        targetRevision: main
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