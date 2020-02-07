# CRD design blueprints

## `XsldCache`
The primary resource which creates the XSLD cache group and the managed statefulset

XSLD REST API: `https://localhost:9445/ibm/api/explorer/#!/cache-member-group`

```yaml
# The xsld cache CRD:
# 1. creates a sateful set of cache instances including data volumes
# 2. creates a headless service
# 3. creates/removes a data grid based on a template
# 4. dynamically joins/removes cache group members
# 5. dynamically exposes nodeport services per each pod for debugging

# TODO
# - Grid template upload
# - PV persistence vs emptyDir
# - resources limits and requests
# - move data grid config to a separate CRD. We can have multiple grids in a group
# - (optional) add separate cache group CRD

# License acceptance (immutable)
license: "accept"

# Number of cache group members
# Size corresponds to the number of the underlying stateful set replicas
# Keeping the default settings for cache memebers, e.g. first 3 memebers are catalog members 
# This is the primary input for the operator reconciliation loop
# API name: numContainers
size: 3

# Cache group name (immutable)
# API name: cacheMemberGroupName
group: "cg1"

# Cache member capacity (Gi) (immutable)
# API name: maxCapacityPerMember
capacity: 1

### Container, Pod and StatefulSet parameters ###

# xsadmin user password (immutable)
# TODO use a secret
adminPwd: "Password123!"

# Cache member secret key (immutable)
# TODO use a secret
secretKey: "Password123!"

# Container and pod settings
image: "ibmcom/xsld:8613"

# TODO:
# Resource limits and requests
#resources:
#  limits:
#    cpu: 2
#  requests:
#    cpu: 1

status:
  # Desired group size
  size: 2
  # Total number of joined members in the cache group
  members: 2
  # Number of online members
  online: 2
```

Custom columns
```
NAME                SIZE   MEMBERS   ONLINE   AG
example-xsldcache   2      1         1        2d
```

## `DataGrid`
Defines the data grid associated with the XsldCache CR

XSLD REST API: `https://localhost:9445/ibm/api/explorer/#!/grid`

```yaml
# Grid name
gridName: "Default"
# Grid description
description: ""
# Grid template object name
templateName: "Simple"
# Number of partitions
numPartition: 3
ownerName: "xsadmin"
useAuthentication: false
useAuthorization: false
transactionTimeout: 30
lockTimeout: 15
numAsyncReplicas: 1
numSyncReplicas: 0
# Evict data from time to live maps
useCapLRUEvictor: false

gridCapacity: null
```

## `DataGridTemplate`
Hold the eXtremeScale Cache data grid template
