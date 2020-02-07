## High availability of XSLD cache

### On pod readiness and clash with XSLD startup

**Issue:** Pod readiness probe (under statefulset) prevents the cache to become ready as the init process fails to resolve its own hostname.

This is a deadlock loop. The pod host isn't DNS-resolvable until the readiness probe succeeds and pod enters Ready state. By Kubernetes design, if the readiness probe fails, the endpoints controller removes the Pod’s IP address from the endpoints of all Services that match the Pod. The statefulset pod hostname is unresolvable if the service endpoint doesn't exist.

Possible solutions:
- Temporary solution is to disable Readiness probe and track the cache status via the main CR.
- Readiness gate? https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate

### On Pod rescheduling

**Issue:** If a statefulset pod is re-scheaduled, its new IP address isn't updated in the cache member information.

It's not clear how the pod IP change affects the cache group memebership and whether the cache member IP must be equal to the actual pod IP. To ensure the IP gets updated should the pod be rescheaduled a workaround is to run a an SQL update in Derby during the container startup after the Derby DB starts. This involved intercepting the default entrypoint script `entrypoint.sh`.

Possible solutions:
- an init container
- modified `entrypoint.sh` with the IP update logic + custom container startup command. Mount as configmap with executable mode (`defaultMode: 0744`). Configmap can be deployed along with the CRDs or via the operator.
- function built into the operator 

