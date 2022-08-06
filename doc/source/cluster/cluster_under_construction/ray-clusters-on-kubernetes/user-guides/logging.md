(kuberay-logging)=

# Logging

This page provides some tips on how to collect logs from your
Ray clusters.

To get started right away,
you can skip to deployment instructions for a sample
configuration.
The rest of this document will provide some background information.

## The Ray log directory
Each Ray pod runs multiple processes.
Ray processes log to files in the directory `/tmp/ray/session_latest/logs`.

## Log processing tools
There are number of log processing tools available within the Kubernetes
ecosystem. This page will cover using Fluent Bit.
Other popular tools include Fluentd, Filebeat, and Promtail.

## Log collection strategies
We mention two strategies for collecting logs written to a pod's filesystem.

**Sidecar containers**
Blah.

**Daemonset**
Blah.

# Setting up logging sidecars with Fluent Bit.
The high-level strategy presented here works with any log aggregation tool.

## Configure log processing
The first step is to create a ConfigMap with configuration
for FluentBit.

Here is minimal configuration for a Fluent sidecar which
* Tails Ray logs
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

Deploy the KubeRay Operator if it's not running yet:
```shell
```

Deploy the Fluent Bit ConfigMap and a single-pod RayCluster with
a FluentBit sidecar
```shell
```

Examine the FluentBit sidecar's STDOUT to see logs for Ray's component processes.
```shell
```
