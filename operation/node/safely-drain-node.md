---
title: "安全迁移节点"
weight: 2
catalog: true
date: 2018-6-23 16:22:24
subtitle:
header-img:
tags:
- Kubernetes
catagories:
- Kubernetes
---

# 1. 迁移Pod

## 1.1. 设置节点是否可调度

确定需要迁移和被迁移的节点，将不允许被迁移的节点设置为不可调度。

```bash
# 查看节点
kubectl get nodes

# 设置节点为不可调度
kubectl cordon <NodeName>

# 设置节点为可调度
kubectl uncordon <NodeName>
```

## 1.2. 执行kubectl drain命令

```bash
# 驱逐节点的所有pod
kubectl drain <NodeName> --force --ignore-daemonsets

# 驱逐指定节点的pod
kubectl drain  <NodeName> --ignore-daemonsets	--pod-selector=pod-template-hash=88964949c
```

示例：

```bash
$ kubectl drain bjzw-prek8sredis-99-40 --force --ignore-daemonsets
node "bjzw-prek8sredis-99-40" already cordoned
WARNING: Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-proxy-bjzw-prek8sredis-99-40; Ignoring DaemonSet-managed pods: calicoopsmonitor-mfpqs, arachnia-agent-j56n8
pod "pre-test-pro2-r-0-redis-2-8-19-1" evicted
pod "pre-test-hwh1-r-8-redis-2-8-19-2" evicted
pod "pre-eos-hdfs-vector-eos-hdfs-redis-2-8-19-0" evicted
```

## 1.3. 特别说明

对于statefulset创建的Pod，kubectl drain的说明如下：

kubectl drain操作会将相应节点上的旧Pod删除，并在可调度节点上面起一个对应的Pod。当旧Pod没有被正常删除的情况下，新Pod不会起来。例如：旧Pod一直处于`Terminating`状态。

对应的解决方式是通过重启相应节点的kubelet，或者强制删除该Pod。

示例：

```bash
# 重启发生`Terminating`节点的kubelet
systemctl restart kubelet

# 强制删除`Terminating`状态的Pod
kubectl delete pod <PodName> --namespace=<Namespace> --force --grace-period=0
```

# 2. kubectl drain 流程图

<img src="https://res.cloudinary.com/dqxtn0ick/image/upload/v1537259779/article/kubernetes/operation/kubectl-drain.svg" width="100%">

# 3. TroubleShooting

1、存在不是通过`ReplicationController`, `ReplicaSet`, `Job`, `DaemonSet` 或者` StatefulSet`创建的Pod（即静态pod，通过文件方式创建的），所以需要设置强制执行的参数`--force`。

```bash
$ kubectl drain bjzw-prek8sredis-99-40
node "bjzw-prek8sredis-99-40" already cordoned
error: unable to drain node "bjzw-prek8sredis-99-40", aborting command...

There are pending nodes to be drained:
 bjzw-prek8sredis-99-40
error: DaemonSet-managed pods (use --ignore-daemonsets to ignore): calicoopsmonitor-mfpqs, arachnia-agent-j56n8; pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet (use --force to override): kube-proxy-bjzw-prek8sredis-99-40
```

2、存在DaemonSet方式管理的Pod，需要设置`--ignore-daemonsets`参数忽略报错。

```bash
$ kubectl drain bjzw-prek8sredis-99-40 --force
node "bjzw-prek8sredis-99-40" already cordoned
error: unable to drain node "bjzw-prek8sredis-99-40", aborting command...

There are pending nodes to be drained:
 bjzw-prek8sredis-99-40
error: DaemonSet-managed pods (use --ignore-daemonsets to ignore): calicoopsmonitor-mfpqs, arachnia-agent-j56n8
```

# 4. kubectl drain

```bash
$ kubectl drain --help
Drain node in preparation for maintenance.

 The given node will be marked unschedulable to prevent new pods from arriving. 'drain' evicts the pods if the API
server supports https://kubernetes.io/docs/concepts/workloads/pods/disruptions/ . Otherwise, it will use normal DELETE
to delete the pods. The 'drain' evicts or deletes all pods except mirror pods (which cannot be deleted through the API
server).  If there are daemon set-managed pods, drain will not proceed without --ignore-daemonsets, and regardless it
will not delete any daemon set-managed pods, because those pods would be immediately replaced by the daemon set
controller, which ignores unschedulable markings.  If there are any pods that are neither mirror pods nor managed by a
replication controller, replica set, daemon set, stateful set, or job, then drain will not delete any pods unless you
use --force.  --force will also allow deletion to proceed if the managing resource of one or more pods is missing.

 'drain' waits for graceful termination. You should not operate on the machine until the command completes.

 When you are ready to put the node back into service, use kubectl uncordon, which will make the node schedulable again.

 https://kubernetes.io/images/docs/kubectl_drain.svg

Examples:
  # Drain node "foo", even if there are pods not managed by a replication controller, replica set, job, daemon set or
stateful set on it
  kubectl drain foo --force

  # As above, but abort if there are pods not managed by a replication controller, replica set, job, daemon set or
stateful set, and use a grace period of 15 minutes
  kubectl drain foo --grace-period=900

Options:
      --chunk-size=500: Return large lists in chunks rather than all at once. Pass 0 to disable. This flag is beta and
may change in the future.
      --delete-emptydir-data=false: Continue even if there are pods using emptyDir (local data that will be deleted when
the node is drained).
      --disable-eviction=false: Force drain to use delete, even if eviction is supported. This will bypass checking
PodDisruptionBudgets, use with caution.
      --dry-run='none': Must be "none", "server", or "client". If client strategy, only print the object that would be
sent, without sending it. If server strategy, submit server-side request without persisting the resource.
      --force=false: Continue even if there are pods not managed by a ReplicationController, ReplicaSet, Job, DaemonSet
or StatefulSet.
      --grace-period=-1: Period of time in seconds given to each pod to terminate gracefully. If negative, the default
value specified in the pod will be used.
      --ignore-daemonsets=false: Ignore DaemonSet-managed pods.
      --ignore-errors=false: Ignore errors occurred between drain nodes in group.
      --pod-selector='': Label selector to filter pods on the node
  -l, --selector='': Selector (label query) to filter on
      --skip-wait-for-delete-timeout=0: If pod DeletionTimestamp older than N seconds, skip waiting for the pod.
Seconds must be greater than 0 to skip.
      --timeout=0s: The length of time to wait before giving up, zero means infinite

Usage:
  kubectl drain NODE [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```

参考文档：

- https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
- https://kubernetes.io/docs/tasks/run-application/configure-pdb/
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain
