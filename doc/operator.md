# Operator functionality

## What the operator does

- provides deployment, scaling, configuration and management of IBM WebSphere eXtreme Scale Liberty Deployment (XSLD) cache on Kubernetes
- operates on custom CRDs: XsldCache, DataGrid, GridTemplate
- creates an XSLD cache group (leveraging a StatefulSet)
- TODO: attaches a data grid configuration via DataGrid CR
- creates a corresponding headless Service (for DNS resolution and service discovery)
- continuously watches the cache group status by polling the cache group API
- joins/disjoins cache memebers to/from the cache group
- mounts a configmap with a custom entrypoint script to provide additional container startup configuration
- deletes persistent volume claims when replicas are removed (cache group scale-down)
- performs rolling image updates; without cache group downtime when Derby replication is configured
 
## How it does things

- leverages Kubernetes StatefulSet controller to manage the state and scaling of the cache group
- uses the [Operator Framework SDK](https://github.com/operator-framework/operator-sdk) to scaffold the operator code base
- uses the [XSLD REST API](https://www.ibm.com/support/knowledgecenter/en/SSTVLU_8.6.1/com.ibm.websphere.extremescale.doc/cxsaccessgridxsldREST.html) to execute cache commands and perform queries
- the REST API Go client package is generated using the [Swagger code generator](https://github.com/swagger-api/swagger-codegen)


## TODO

- [ ] Cache readiness state to XsldCache status
- [x] DataGrid CRD - reconciliation, status watch, Error state recovery
- [ ] GridTemplate CRD 
- [ ] replace Tasksi Status by Cache Conditions - member ready, watch is running, dis/joining. see https://github.com/coreos/etcd-operator/blob/master/pkg/apis/etcd/v1beta2/status.go
- [ ] Setup Derby replication when 2+ members, switch Master for API access, Status monitoring - retry with new Master
- [x] Setting number of catalog servers (first x members will act as catalog servers)
- [ ] Add volume storage class settings into the CRD
- [ ] Pod spec settings field into the XsldCache
    * [ ] ServiceAccout
    * [ ] PullSecret
    * [ ] scheaduling anti-affinity - no two cache instances on the same node
- [ ] unify logging - log levels 
- [ ] Makefile
- [ ] Helm - deploy operator and an instance
- [ ] tests


## Test cases

Cache scaling:

- Create resource, 2 replicas (automatic member join operation), add data grid
- Disjoin member
- Join memeber
- Scale up by 2, scale down by 2

Failure states:

- random container restart
- pod failure and re-schedule (pod delete)
- 2 pods fail/restart
- Primary (pod-0) restart/delete - check status using the remaining memebers
- failure in the middle of a join operation
- failure inthe middle of a disjoin operation
- operator pod restart
