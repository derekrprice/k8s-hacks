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
and probably some other tickets that aren't properly marked as duplicates yet.  Basically, K8s is leaving orphaned mounts behind when pods are cleaned up, which causes CPU usage to linearly increase until the whole node is taken down.  This is particularly exacerbated by K8s CronJobs since they have such a short lifecycle.  With a few cronjobs running every minute, I am able to take down a two-core AKS cluster node in about a week.  Depending on which ticket you look at, this problem is attributed to a failure of K8s to cleanup mounts or a bad interaction between certain kernel versions and certain versions of systemd.  My money is on the latter since, on my system, the pod directories are already gone but systemd still has the cgroup watches registered.

Anyhow, this DaemonSet works around the problem by running a script hourly to search for the orphaned mounts and asking systemd to stop watching them.

### Installing
kubectl apply -f systemd-cgroup-gc.yaml
