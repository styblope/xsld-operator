# xsld-operator
This XSLD Operator is a Kubernetes operator for managing [IBM eXtreme Scale Liberty Deployment (XSLD)](https://www.ibm.com/support/knowledgecenter/en/SSTVLU_8.6.1/com.ibm.websphere.extremescale.doc/cxsgetstartxsld.html) based on the [official IBM XSLD container image](https://hub.docker.com/r/ibmcom/xsld/).

> **Warning:** This project is a result of a customer proof-of-concept and it isn't polished enough for actual production use. At this stage, the binary will only be provided upon individual requests.

The XSLD Operator is written in Go using the open-source [Operator-SDK](https://github.com/operator-framework/operator-sdk) which is part of the [Operator Framework](https://github.com/operator-framework) toolkit maintained by RedHat. In terms of the Operator Framework maturity model, the operator roughly covers the following phases at various levels of completeness:
<p align="center">
<img src="doc/images/operator-phases.png" width=600>
</p>

- [x] **Basic install**  
The operator provides a user-friendly deployment of XSLD cache on Kubernetes. Using custom CRDs aligned with the inner logical structure of XSLD, users can declaratively create new multi-member XSLD cache deployments and apply custom Data Grid configurations without additional manual configuration inside the containers. The operator also automatically provisions and manages per-member persistent storage throughout the whole cache deployment lifecycle including scaling. Out-of-band adjustments to the XSLD container startup are devised in order to cater for the specifics of Kubernetes networking (see below).

- [x] **Seamless Upgrades**  
The XSLD operator builds on a standard Kubernetes StatefulSet replication controller and image rolling update is enabled via the main CRD.

- [x] **Full lifecycle**  
Aside of automatic persistent storage provisioning and de-provisioning, no further data backup/recovery measures are implemented due to the nature of the in-memory cache.

- [ ] **Deep Insights**  
This area yet to be explored and addressed, possibly exposing and integrating the built-in XSLD performance metrics and logs.

- [x] **Auto Pilot**  
The core functionality of the XSLD operator lies in the automated XSLD cache group establishment, maintenance, dynamic scaling and data partition provisioning and configuration. In particular, XSLD cache group rescaling involves highly coordinated triggering of join/disjoin tasks executed on the cache members using the XSLD REST APIs.

### About eXtreme Scale Liberty Deployment

The WebSphere® eXtreme Scale Liberty Deployment provides Liberty caching servers, caching operation tools, an administration console, and the out-of-the-box REST administration services based on the latest industry standards and specifications. XSLD is built based on the core eXtreme Scale technology.

[WebSphere eXtreme Scale Liberty Deployment (XSLD)](https://www.ibm.com/support/knowledgecenter/en/SSTVLU_8.6.1/com.ibm.websphere.extremescale.doc/cxsgetstartxsld.html) provides more features and extensions such as cloud enablement, speed of deployment, ease of administration, and ease of integration, and customizable scripts and REST APIs. XSLD runs within the IBM® Liberty runtime environment. The main components are:

- WebSphere eXtreme Scale caching service and servers
- Administrative REST services
- Operational REST services
- Multi-tenancy processes

## How the XSLD Operator works
The operator design follows closely the concepts and best practises provided by the Operator SDK project. *Custom Resource Definitions (CRD)* are defined for setting up the multi-pod XSLD cache group and configuration of the data grid partitions inside the cache group. A controller defined around each of the CRD implements the reconciliation logic to maintain the desired state.

The operator leverages and extends the default Kubernetes *StatefulSet* replication controller with managed *PersistentVolumeClaims* backed by *PersistentVolumes*, currently on NFS. Each PVC holds data and configuration associated with the respective XSLD cache instance. The PVCs lifetime is aligned with the desired state of the cache group replicas, which ensures the XSLD cache group is recovered upon pod or container restarts/reschedules.  Unlike the default StatefulSet mode, the PVC are deleted and the data volumes are reclaimed back as soon as pods are removed from the managed set as a result of a scale-down request.

The XSLD cache configuration, control and status retrieval is performed through the [XSLD Admin and Operational REST API](https://www.ibm.com/support/knowledgecenter/en/SSTVLU_8.6.1/com.ibm.websphere.extremescale.doc/cxsaccessgridxsldREST.html). [Swagger Codegen](https://github.com/swagger-api/swagger-codegen) generator is used to generate the corresponding Go API client module.

## Custom Resource Definitions
The main **`XsldCache`** CRD defines a group of interconnected XSLD members. When an `XsldCache` resource is created the controller creates a corresponding StatefultSet and initializes the cache members in the ordinal sequence, allocating new PVCs as data volumes. New members are automatically joined to the cache group by execution of the XSLD Join Task. State logic is in place which triggers the join/disjoin tasks in the correct phases of the pod startup/termination taking care of request retries and parallel task conflicts. The controller maintains an independent loop (via a goroutine) for watching the runtime cache status.

The `XsldCache` resource controller also creates a matching Headless Service for the StatefulSet which ensures that pods are DNS/resolvable within the cluster (the XSLD requires FQDN hostname resolution).

When the XSLD cache group is scheduled for a scale-down the leaving member correctly disjoins (via the API Disjoin Task) the group prior to the pod termination. After the pod terminates, the controller deletes the associated PVC (which isn't done by default by the StatefulSet controller)

### `XsldCache`
**`XsldCache`** is the primary resource which defines the XSLD cache group and the managed StatefulSet

```yaml
apiVersion: xsld.ibm.com/v1alpha1
kind: XsldCache
metadata:
  name: example-xsldcache
spec:
  # License acceptance (immutable)
  license: accept
  # Size defines the desired number of members joined in the
  # cache group. Corresponds to the number of the stateful set replicas
  size: 1
  # Image is the repository and tag of the XSLD container image
  # Default is `ibmcom/xsld:latest`
  image: ibmcom/xsld:8613
  # AdminPwd is the xsadmin user password (immutable). 
  adminPwd: Password123!
  # SecretKey is the secret key for the cache group (immutable).
  secretKey: Password123!
  # Capacity is memory size allocated to a member in GB (immutable).
  # Default capacity is 1GB
  capacity: 1 
  # Catalogs is the number of catalog servers in the cache
  # group. Default is 3 catalog servers.
  catalogs: 3
  # EntrypointConfigMap is the name of a configmap with custom
  # entrypoint.sh which overides the default entrypoint
  entrypointConfigMap: ip-update-entrypoint
  # Group is the name of the cache group (immutable). Default
  # name is 'cg1'
  group: cg1
 ```

Status fields
```
NAME                MEMBERS   ONLINE   TASKS   GRIDS   AGE
example-xsldcache   2         2        1       1       2d
```

### `DataGrid`

**`DataGrid`** defines the data grids that are to be provisioned inside the matching `XsldCache` instance in the given namespace. Multiple data grids, i.e. multiple `DataGrid` objects, can be created in a cache.

```yaml
apiVersion: xsld.ibm.com/v1alpha1
kind: DataGrid
metadata:
  name: example-datagrid
spec:
  # Description of the grid
  decription:
  # Grid capacity in MB default value 0 means unlimited
  gridCapacity:
  # Type of grid: Simple, Session, DynaCache, Custom
  gridType: Simple
  # Name of the group which owns the grid this field is overriden
  #  by the name defined in the referenced XsldCache object
  groupName:
  # Lock time out for the backing maps in seconds default is
  lockTimeout: 15
  # Number of asynchronous replicas default is 1 replica
  numAsyncReplicas: 1
  # Number of partitions the number must be a prime number. Default is 3
  numPartition: 3
  # Number of synchronous replicas
  numSyncReplicas:
  # Name of tde grid owner default is xsadmin
  ownerName: xsadmin
  # Grid template name
  templateName: Simple
  # How long each map entry is present in seconds
  timeToLive:
  # Transaction timeout in seconds default is 30 seconds
  transactionTimeout: 30
  # Turn on authentication
  useAuthentication: false
  # Turn on authorization
  useAuthorization: false
  # Use LRU evictor for grid capacity
  useCapLRUEvictor:
```

`DataGrid` Status returns the online status of the data grid

## Usage

The following commands assume the operator is already running either in the cluster as a deployment or locally using the `operator-sdk up local` mode. The operator deployment details are not covered in this overview.

#### Create the xsld-operator CRDs in Kubernetes
```sh
$ kubectl create -f deploy/crds/xsld.ibm.com_xsldcaches_crd.yaml
$ kubectl create -f deploy/crds/xsld.ibm.com_datagrids_crd.yaml
$ kubectl api-resources | grep xsld

NAME                              SHORTNAMES   APIGROUP                                      NAMESPACED   KIND
datagrids                                      xsld.ibm.com                                  true         DataGrid
xsldcaches                                     xsld.ibm.com                                  true         XsldCache
```

#### Create new `XsldCache` resource in the current context namespace
```sh
apiVersion: xsld.ibm.com/v1alpha1
kind: XsldCache
metadata:
  name: example-xsldcache
spec:
  license: accept
  size: 1
  adminPwd: Password123!
  secretKey: Password123!
  image: ibmcom/xsld:8613
  entrypointConfigMap: ip-update-entrypoint
```

It takes several minutes for the XSLD processes to initialize. The operator watches the cache runtime and updates its status when the member is up and running.
```sh
$ kubectl create -f deploy/crds/xsld.ibm.com_v1alpha1_xsldcache_cr.yaml
$ kubectl get xsldcache
NAME                                       MEMBERS   ONLINE   TASKS   GRIDS   AGE 
xsldcache.xsld.ibm.com/example-xsldcache   1         1                        1d
```

Check the actual Kubernetes objects created and managed by the operator: `StatefultSet`, headless `Service`, `PVCs`, `Pods`
```sh
$ kubectl get all -l app=example-xsldcache
NAME                      READY   STATUS    RESTARTS   AGE
pod/example-xsldcache-0   1/1     Running   0          165m

NAME                        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/example-xsldcache   ClusterIP   None         <none>        9443/TCP   165m

NAME                                 READY   AGE
statefulset.apps/example-xsldcache   2/2     165m

$ kubectl get pvc
NAME                      STATUS   VOLUME                    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
vol-example-xsldcache-0   Bound    example-xsldcache-vol-1   100Mi      RWO                           166m
```

#### Scale up cache to form a group of 2 members.
The operator first scales the underlying `StatefulSet` and allocates a new `PVC`. After the new container is initialized the operator runs an API Join Task which joins the new member to the group. The group enrollment process is captured in the `XsldCache` resource Status. Note, the XSLD member startup and join tasks take up to 7 minutes!
```sh
$ kubectl edit xsldcache example-xsldcache # Set "scale" field from 1 to 2
$ watch kubectl get xsldcache,pod,pvc
```
![](doc/images/up-scale.svg)

The cache group is formed and both instances are online.

```
NAME                MEMBERS   ONLINE   TASKS   GRIDS   AGE
example-xsldcache   2         2                        1d
```

#### Create new `DataGrid` resource
```yaml
apiVersion: xsld.ibm.com/v1alpha1
kind: DataGrid
metadata:
  name: example-datagrid
spec:
  # Description of the grid
  # description:
  # Name of the grid owner
  # ownerName: xsadmin
  # Turn on authentication
  # useAuthentication: false
  # Turn on authorization
  # useAuthorization: false
  # Grid capacity in MB
  # gridCapacity: 
  # Use LRU evictor for grid capacity
  # useCapLRUEvictor:
  # Transaction timeout in seconds
  # transactionTimeout:
  # Number of partitions
  numPartition: 3
  # Number of synchronous replicas
  # numSyncReplicas:
  # Number of asynchronous replicas
  # numAsyncReplicas:
  # How long each map entry is present in seconds
  # timeToLive:
  # Lock time out for the backing maps in seconds
  # lockTimeout:
  # Type of grid
  # gridType: Simple
  # Grid template name 
  templateName: Simple
  # Name of the group which owns the grid
  # groupName:
```

```bash
$ kubectl create -f deploy/crds/xsld.ibm.com_v1alpha1_datagrid_cr.yaml
$ kubectl get datagrid
NAME               TYPE   CAPACITY   STATUS
example-datagrid
```

Watch the group being added to the cache
```sh
$ watch kubectl get xsldcache example-xsldcache
```
![](doc/images/create-grid.svg)

#### Cache scale-down
```sh
$ kubectl edit xsldcache example-xsldcache # set "size" field  back to 1
$ watch kubectl get xsldcache,pod,pvc
```
![](doc/images/down-scale.svg)


#### Verify the cache configuration in the XSLD WebUI
<div align="center">
<img src="doc/images/screenshot1.png">
<p></p>
<img src="doc/images/screenshot2.png">
<p></p>
<img src="doc/images/screenshot3.png">
</div>
