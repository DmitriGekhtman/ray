(kuberay-logging)=

# Logging

This page provides some tips on how to collect logs from your
Ray clusters.

:::{tip}
Skip to {ref}`the deployment instructions<kuberay-logging-tldr>`
for a sample configuration showing how to extract logs from a Ray pod.
:::

## The Ray log directory
Each Ray pod runs multiple processes.
Ray processes log to files in the directory `/tmp/ray/session_latest/logs`.

## Log processing tools
There are number of log processing tools available within the Kubernetes
ecosystem. This page will cover using [Fluent Bit][FluentBit].
Other popular tools include [Fluentd][Fluentd], [Filebeat][Filebeat], and [Promtail][Promtail].

## Log collection strategies
We mention two strategies for collecting logs written to a pod's filesystem,
**sidecar containers** and **daemonsets**. You can read more about these logging
patterns in the [Kubernetes documentation][KubDoc].

### Sidecar containers
We will provide an {ref}`example<kuberay-fluentbit>` of this strategy in this guide.
You can process logs by specifying an appropriate log-processing **sidecar**
for each Ray pod. Ray containers should be configured to share the `/tmp/ray`
directory with the logging sidecar via a volume mount.

You can configure the sidecar to either
* stream Ray logs to the sidecar's STDOUT
* export logs to an external service

### Daemonset
Alternatively, it is possible to collect logs at the Kubernetes node level.
To do this, one deploys a log-processing daemonset on a subset of
the Kubernetes Nodes in your cluster. With this strategy, it is key to mount
the Ray container's `/tmp/ray` directory to the appropriate `hostPath`.

(kuberay-fluentbit)=
# Setting up logging sidecars with Fluent Bit.
In this section, we give a concrete example of how to set up a log-emitting
[Fluent Bit][FluentBit] sidecar for a Ray pod.

## Configure log processing
The first step is to create a ConfigMap with configuration
for FluentBit.

Here is a minimal ConfigMap for a Fluent sidecar which
* Tails Ray logs.
* Outputs the logs to the container's STDOUT.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentbit-config
data:
  fluent-bit.conf: |
    [INPUT]
        Name tail
        Path /tmp/ray/session_latest/logs/*
        Tag ray
        Path_Key true
        Refresh_Interval 5
    [OUTPUT]
        Name stdout
        Match *
```
In addition to streaming logs to stdout, you can export logs to any
[storage backend][FluentBitStorage] supported by FluentBit.

## Add logging sidecars to your RayCluster CR.

### Add log and config volumes.
For each pod template in our RayCluster CR, we will
need to add two volumes: One volume for Ray's logs
and another volume to store Fluent Bit configuration from the ConfigMap
applied above:
```yaml
volumes:
- name: ray-logs
  emptyDir: {}
- name: fluentbit-config
  configMap:
    name: fluentbit-config
```

### Mount the Ray log directory
Add the following volume mount to the Ray container's configuration.
```yaml
volumeMounts:
- mountPath: /tmp/ray
  name: ray-logs
```

### Add the Fluent Bit sidecar
Finally, add the Fluent Bit sidecar container to each pod configuration
in your RayCluster CR:
```yaml
- name: fluentbit
  image: fluent/fluent-bit:1.9.6
  volumeMounts:
  - mountPath: /tmp/ray
    name: ray-logs
  - mountPath: /fluent-bit/etc/fluent-bit.conf
    name: fluentbit-config
```
Mounting the `ray-logs` volume gives the sidecar container access to Ray's logs.
The `fluentbit-config` volume gives the sidecar access to logging configuration.

(kuberay-logging-tldr)=
## Putting everything together

Deploy the KubeRay Operator if you haven't yet.
```shell
git clone https://github.com/ray-project/kuberay -b release-0.3
kubectl create -k kuberay/ray-operator/config/default
```

Deploy the Fluent Bit ConfigMap and a single-pod RayCluster with
a Fluent Bit sidecar.
```shell
# Starting from the parent of cloned Ray master.
pushd ray/doc/source/cluster/cluster_under_construction/ray-clusters-on-kubernetes/configs/
kubectl apply -f ray-cluster.log.yaml
popd
```

Determine the Ray pod's name with
```shell
kubectl get pod | grep raycluster-complete-logs
```

Examine the FluentBit sidecar's STDOUT to see logs for Ray's component processes.
```shell
# Substitute the name of your Ray pod.
kubectl logs raycluster-complete-logs-head-xxxxx -c fluentbit
```

[FluentBit]: https://docs.fluentbit.io/manual
[FluentBitStorage]: https://docs.fluentbit.io/manual
[Filebeat]: https://www.elastic.co/guide/en/beats/filebeat/7.17/index.html
[Fluentd]: https://docs.fluentd.org/
[Promtail]: https://grafana.com/docs/loki/latest/clients/promtail/
[KubDoc]: https://kubernetes.io/docs/concepts/cluster-administration/logging/
