## Build and testing

TODO:

- Makefile

### Create project
Tightly based on [operator-sdk user guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md)

```sh
operator-sdk new xsld-operator --repo=github.com/styblope/xsld-operator
cd xsld-operator
```

### Add Custom Resource Definitions
```
# XSLD Cache Member Group  (primary CRD)
operator-sdk add api --api-version=xsld.ibm.com/v1alpha1 --kind=XsldCache
# XSLD Data Grid 
operator-sdk add api --api-version=xsld.ibm.com/v1alpha1 --kind=DataGrid
# Grid Template
operator-sdk add api --api-version=xsld.ibm.com/v1alpha1 --kind=GridTemplate
```

After modifying the `*_types.go` run
```
operator-sdk generate k8s
```

To update the OpenAPI validation section in the CRD `deploy/crds/cache.example.com_memcacheds_crd.yaml` run
```
operator-sdk generate openapi
```

### Add Controller
```
operator-sdk add controller --api-version=xsld.ibm.com/v1alpha1 --kind=XsldCache
operator-sdk add controller --api-version=xsld.ibm.com/v1alpha1 --kind=DataGrid
operator-sdk add controller --api-version=xsld.ibm.com/v1alpha1 --kind=GridTemplate
```

### Build and run the operator

Register CRDs with Kubernetes
```
kubectl create -f deploy/crds/xsld.ibm.com_xsldcaches_crd.yaml
```

Create the example `xsldCache` CR
```
kubectl create -f deploy/crds/xsld.ibm.com_v1alpha1_xsldcache_cr.yaml
```

Run operator locally outside the cluster with the default kubeconfig `$HOME/.kube/config`. This is only good for development and testing.
```sh
export OPERATOR_NAME=xsld-operator
operator-sdk up local --namespace=default
```

**Note:** Make sure to establish pod localhost port forwarding (`kubectl port-forwared <pod> 9445:9445 9443:9443`) and service FQDN name lookups.
