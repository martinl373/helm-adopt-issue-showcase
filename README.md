# CRD Instance Resources and Helm Adopt Issue

## 1. Install CRDs
These CRDs are from the Kyverno service. We do not install Kyverno nor use it
in any way. We are only borrow one cluster-scoped and one namespace-scoped CRD
from it as a test. (Other CRDs have been tested, with the same results)

```bash
kubectl create -f test-setup/crds/sample-custom-crds.yaml
```

## Create a test namespace
Create a namespace for all the namespaced test resources and the helm release.

```bash
kubectl create namespace test-namespace
```

## 2. Install the Initial versions of test resources

```bash
for file in test-setup/manifests/*; do
    kubectl create -f ${file}
done
```

## 3. Try to run the helm chart now
This should fail with Helm complaining that it does not own the files.

```bash
helm upgrade --install --namespace test-namespace test-release test-chart/
```

## 4. Ready all resources for Helm adoption
This is a official feature of helm since version 3.2ish. The adoption is really quite
simple, just adding a label and some annotations. (See the `helm-adopt.sh` for details)

```bash
scripts/helm-adopt.sh clusterpolicy.kyverno.io test-clusterpolicy test-release test-namespace
scripts/helm-adopt.sh policy.kyverno.io test-policy test-release test-namespace
scripts/helm-adopt.sh clusterrole.rbac.authorization.k8s.io test-clusterrole test-release test-namespace
scripts/helm-adopt.sh role.rbac.authorization.k8s.io test-role test-release test-namespace
scripts/helm-adopt.sh configmap test-configmap test-release test-namespace
```

## 5. Run Helm again (with resources marked for adoption)
Now all will seem to run fine, Helm will not complain, but some of the resources
will just be silently ignored. This can be run multiple times, these files will
always be ignored. The only way to get helm to update these files is to change
them in the chart AFTER the adoption and first helm upgrade/install is run.
Basically there needs to be a diff between the chart and the manifests in the
Helm release secret for the changes to actually apply. But as can be seen in the
table below, this is only true for resources defined by a CRD. Native k8s resource
work fine.

```bash
helm upgrade --install --namespace test-namespace test-release test-chart/
```

Looking at the output from Helm in debug mode (Adding the `--debug` flag) one can
see that helm actually thinks that these files are not different (in cluster vs chart)

```text
client.go:396: [debug] checking 5 resources for changes
client.go:684: [debug] Patch ConfigMap "test-configmap" in namespace test-namespace
client.go:684: [debug] Patch ClusterRole "test-clusterrole" in namespace
client.go:684: [debug] Patch Role "test-role" in namespace test-namespace
client.go:675: [debug] Looks like there are no changes for ClusterPolicy "test-clusterpolicy"
client.go:675: [debug] Looks like there are no changes for Policy "test-policy"
```

If the `helm upgrade --install` command is repeated multiple times, helm will
(from the second run and onwards) recognize that the CRD based resources are
indeed different, but it will still not actually change them.

```text
client.go:396: [debug] checking 5 resources for changes
client.go:675: [debug] Looks like there are no changes for ConfigMap "test-configmap"
client.go:675: [debug] Looks like there are no changes for ClusterRole "test-clusterrole"
client.go:675: [debug] Looks like there are no changes for Role "test-role"
client.go:684: [debug] Patch ClusterPolicy "test-clusterpolicy" in namespace
client.go:684: [debug] Patch Policy "test-policy" in namespace test-namespace
```

## Results
This is the results from the first run of `helm upgrade --install` with target
resources marked for adoption in the cluster. (The "Changed in cluster?" column
denotes if a resource of that type did change in cluster in accordance with the
helm chart)

| Resource Type | Changed in cluster? |
|---|---|
| clusterpolicy.kyverno.io  | false |
| policy.kyverno.io         | false |
| clusterrole.rbac.authorization.k8s.io | true |
| role.rbac.authorization.k8s.io | true |
| configmap | true |
