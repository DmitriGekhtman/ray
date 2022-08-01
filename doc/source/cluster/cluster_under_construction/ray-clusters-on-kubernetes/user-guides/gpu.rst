.. _kuberay-gpu:

Using GPUs
==========
This document provides some tips on GPU usage with Kubernetes.

To use GPUs on Kubernetes, you will need to configure both your Kubernetes setup and add additional values to your Ray cluster configuration.

For relevant documentation for GPU usage on different clouds, see instructions for `GKE`_, for `EKS`_, and for `AKS`_.

The `Ray Docker Hub <https://hub.docker.com/r/rayproject/>`_ hosts CUDA-based images packaged with Ray for use in Kubernetes pods.
For example, the image ``rayproject/ray-ml:2.0.0-gpu`` is ideal for running GPU-based ML workloads with Ray 2.0.0.
Read :ref:`here<docker-images>` for further details on Ray images.

Using Nvidia GPUs requires specifying the relevant resource `limits` in the container fields of your Kubernetes configurations.
(Kubernetes `sets <https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins>`_
the GPU request equal to the limit; you may want to specify requests for purposes of documentation.)
Here is a configurations snippet for a RayCluster workerGroup of up
to 5 GPU workers.

.. code-block:: yaml

   groupName: gpu-group
   replicas: 0
   minReplicas: 0
   maxReplicas: 5
   ...
   template:
       spec:
        ...
        containers:
         - name: ray-node
           image: rayproject/ray-ml:2.0.0-gpu
           ...
           resources:
            cpu: 3
            memory: 50Gi
            nvidia.com/gpu: 1 # Optional, included just for documentation.
           limits:
            cpu: 3
            memory: 50Gi
            nvidia.com/gpu: 1 # Required to use GPU.

Each of the Ray pods in the group can be scheduled on an AWS `p2.xlarge` instance (1 GPU, 4vCPU, 61Gi RAM).
GPU instances are expensive -- consider setting up autoscaling for your GPU Ray workers, as suggested in the example above.

GPUs and Ray
____________

GPU resources specified in a workerGroup's Ray container resource limits will be advertised to
the Ray scheduler and Ray autoscaler.

GPU workload scheduling
~~~~~~~~~~~~~~~~~~~~~~~
After a Ray pod with access to GPU is deployed, it will
be able to execute tasks and actors decorated with `@ray.remote(num_gpus=1)`.

GPU autoscaling
~~~~~~~~~~~~~~~
The Ray autoscaler is aware of each Ray worker group's GPU capacity.
Say we have a RayCluster configured as above:
- We have a worker group of Ray pods with 1 unit of GPU capacity each
- The Ray cluster does not currently have any workers from that group
- `maxReplicas` for the group is at least 2

Then the following Ray program will trigger upscaling of 2 GPU workers.
```python
import ray

ray.init()

@ray.remote(num_gpus=1)
class GPUActor:
    def say_hello(self):
        print("I live in a pod with GPU access.")

gpu_actors = [GPUActor.remote() for _ in range(2)]
ray.get([actor.say_hello.remote() for actor in gpu_actors])
```
After the program exits, the actors will be garbage collected.
The GPU worker pods will then be scaled down after the idle timeout (60 seconds by default).
If the GPU worker pods were running on an autoscaling pool of Kubernetes nodes, the Kubernetes
nodes will be scaled down as well.
(link)

You can also make a direct request to the autoscaler scale up GPU resources.
(link)
```python
import ray

ray.init()
ray.autoscaler.sdk.request_resources(bundles=[{"GPU": 1} * 2])
```
After the nodes are scaled up, they will persist until the request is explicitly overridden.
The following program will remove the resource request.
```python
import ray

ray.init()
ray.autoscaler.sdk.request_resources(bundles=[])
```
The GPU workers can then scale down.

Overriding Ray GPU capacity (advanced)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
For specialized use-cases, it is possible to override the GPU capacity advertised to Ray.
To do so, set a value for the `num-gpus` key of the head or worker group's `rayStartParams`.
For example,
```yaml
    rayStartParams:
        num-gpus: "2"
```
The Ray scheduler and autoscaler will then account 2 units of GPU capacity for each
Ray pod in the group, even if the container limits do not indicate the presence of GPU.

GPU taints and tolerations
--------------------------
.. note::

  Users using a managed Kubernetes service probably don't need to worry about this section.

The `Nvidia gpu plugin`_ for Kubernetes applies `taints`_ to GPU nodes; these taints prevent non-GPU pods from being scheduled on GPU nodes.
Managed Kubernetes services like GKE, EKS, and AKS automatically apply matching `tolerations`_
to pods requesting GPU resources. Tolerations are applied by means of Kubernetes's `ExtendedResourceToleration`_ `admission controller`_.
If this admission controller is not enabled for your Kubernetes cluster, you may need to manually add a GPU toleration each of to your GPU pod configurations. For example,

.. code-block:: yaml

  apiVersion: v1
  kind: Pod
  metadata:
   generateName: example-cluster-ray-worker
   spec:
   ...
   tolerations:
   - effect: NoSchedule
     key: nvidia.com/gpu
     operator: Exists
   ...
   containers:
   - name: ray-node
     image: rayproject/ray:nightly-gpu
     ...

Further reference and discussion
--------------------------------
Read about Kubernetes device plugins `here <https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/>`__,
about Kubernetes GPU plugins `here <https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus>`__,
and about Nvidia's GPU plugin for Kubernetes `here <https://github.com/NVIDIA/k8s-device-plugin>`__.

.. _`GKE`: https://cloud.google.com/kubernetes-engine/docs/how-to/gpus
.. _`EKS`: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
.. _`AKS`: https://docs.microsoft.com/en-us/azure/aks/gpu-cluster

.. _`tolerations`: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
.. _`taints`: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
.. _`Nvidia gpu plugin`: https://github.com/NVIDIA/k8s-device-plugin
.. _`admission controller`: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
.. _`ExtendedResourceToleration`: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration
