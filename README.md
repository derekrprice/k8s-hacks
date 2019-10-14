# k8s-hacks
Scripts Used to Work Around K8s Problems

I'm attaching some code here so that I can share it.
## systemd-cgroup-gc.yaml
This works around:
* https://github.com/kubernetes/kubernetes/issues/64137
* https://github.com/Azure/AKS/issues/750
* https://github.com/kubernetes/kubernetes/pull/73799
* https://github.com/kubernetes/kubernetes/issues/60987
* https://github.com/rancher/k3s/issues/294
* https://github.com/kubernetes/kubernetes/issues/70324

and probably some other tickets that aren't properly marked as duplicates yet.  Basically, K8s is leaving orphaned mounts behind when pods are cleaned up, which causes CPU usage to linearly increase until the whole node is taken down.  This is particularly exacerbated by K8s CronJobs since they have such a short lifecycle.  With a few cronjobs running every minute, I am able to take down a two-core AKS cluster node in about a week.  Depending on which ticket you look at, this problem is attributed to a failure of K8s to cleanup mounts or a bad interaction between certain kernel versions and certain versions of systemd.  My money is on the latter since, on my system, the pod directories (and, thus their mount subdirs) are already gone but systemd still has the cgroup watches registered.

Anyhow, this DaemonSet works around the problem by running a script
hourly to search for the orphaned mounts and asking systemd to stop
watching them.

### Installing
kubectl apply -f systemd-cgroup-gc.yaml

### Uninstalling
kubectl delete -f systemd-cgroup-gc.yaml

## node-sysctl.yaml
Simple Daemonset to set sysctl settings on nodes.  Right now it
only sets a higher limit for `fs.inotify.max_user_watches`.  I
think that the only reason that we are encountering watch limit
issues right now is the same bugs that the `system-cgroup-gc`
package is working around.

### Installing
kubectl apply -f node-sysctls.yaml

### Uninstalling
kubectl delete -f node-sysctls.yaml

## reboot-regularly.yaml
Simple Daemonset to reboot nodes regularly.  Uses
[kured](https://github.com/weaveworks/kured) to safely trigger a
rolling reboot during a specified window, if the node has not been
rebooted within a specified timeframe.  If kured is not running,
then marking the node for reboot will have no effect.

### Installing
kubectl apply -f regular-reboot.yaml

### Uninstalling
kubectl delete -f regular-reboot.yaml
