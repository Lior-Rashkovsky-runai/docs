Below are the prerequisites of a cluster installed with Run:ai. 

## Software Requirements
### Kubernetes

Run:ai requires Kubernetes. The latest Run:ai version supports Kubernetes versions 1.21 through 1.24. For RedHat OpenShift. Run:ai supports OpenShift 4.8 to 4.10.

Run:ai has been tested with the following Kubernetes distributions: 

| Target Platform                          | Description | Installation Notes | 
|------------------------------------------|-------------|--------------------|
| Vanilla Kubernetes                       |  Using no specific distribution but rather k8s native installation  | |
| EKS | Amazon Elastic Kubernetes Service  | |
| AKS | Azure Kubernetes Services          | |
| GKE | Google Kubernetes Engine           | GKE has a different software stack for NVIDIA. To install Run:ai on GKE please contact customer support. | 
| OCP | OpenShift Container Platform       | The Run:ai operator is [certified](https://catalog.redhat.com/software/operators/detail/60be3acc3308418324b5e9d8){target=_blank} for OpenShift by Red Hat. | 
| RKE | Rancher Kubernetes Engine          | When installing Run:ai, select _On Premise_. You must perform the mandatory extra step [here](../../troubleshooting/troubleshooting.md#kubernetes-specific-issues). RKE2 has a defect which requires a specific installation. Please contact Run:ai customer support for additional details. |
| Ezmeral | HPE Ezmeral Container Platform | See Run:ai at [Ezmeral marketplace](https://www.hpe.com/us/en/software/marketplace/runai.html){target=_blank}  |
| Tanzu | VMWare Kubernetes | Tanzu supports _containerd_ rather than _docker_. See the NVIDIA prerequisites below as well as [cluster customization](customize-cluster-install.md) for changes required for containerd |
| Canonical Kubernetes | a.k.a Charmed Kubernetes | | 

A full list of Kubernetes partners can be found here: [https://kubernetes.io/docs/setup/](https://kubernetes.io/docs/setup/){target=_blank}. In addition, Run:ai provides instructions for a simple (non-production-ready) [Kubernetes Installation](install-k8s.md).


!!! Notes
    * Kubernetes [recommends](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/){target=_blank} the usage of the `systemd` as the [container runtime cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker){target=_blank}. Kubernetes 1.22 and above defaults to `systemd`. 
    * Run:ai Supports Kubernetes [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/){target=_blank} if used. 
### NVIDIA 

=== "On Prem"    
    Follow the [Getting Started guide](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#install-nvidia-gpu-operator){target=blank} to install the __NVIDIA GPU Operator__ version 1.9 or higher. 

=== "EKS"
    * Do not install the NVIDIA device plug-in  (as we want the NVIDIA GPU Operator to install it instead). When using the [eksctl](https://eksctl.io/){target=_blank} tool to create an AWS EKS cluster, use the flag `--install-nvidia-plugin=false` to disable this install.
    * Follow the [Getting Started guide](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#install-nvidia-gpu-operator){target=blank} to install the __NVIDIA GPU Operator__ version 1.9 or higher. For GPU nodes, EKS uses an AMI which already contains the NVIDIA drivers. As such, you must use the GPU Operator flags: `--set driver.enabled=false --set toolkit.enabled=false --set migManager.enabled=false`. 

=== "GKE"
    Google provides a different method for installing NVIDIA Device drivers and the NVIDIA device plug-in. Detailed [here](https://cloud.google.com/kubernetes-engine/docs/how-to/gpus#installing_drivers){target=_blank}. As such, using the GPU Operator is not an option. You will need to manually install the [Kubernetes Node feature discovery](https://github.com/kubernetes-sigs/node-feature-discovery){target=_blank}, [NVIDIA GPU feature discovery](https://github.com/NVIDIA/gpu-feature-discovery){target=_blank} and [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter){target=_blank}. The installation is not trivial and we recommend contacting Run:ai customer support. 

=== "RKE"
    Install the __NVIDIA GPU Operator__ as discussed [here](https://thenewstack.io/install-a-nvidia-gpu-operator-on-rke2-kubernetes-cluster/){target=_blank}.

!!! Notes
    * Use the default namespace `gpu-operator`. Otherwise, you must specify the target namespace using the flag `runai-operator.config.nvidiaDcgmExporter.namespace` as described in [customized cluster installation](customize-cluster-install.md).
    * NVIDIA drivers may already be installed on the nodes. In such cases, use the NVIDIA GPU Operator flags `--set driver.enabled=false --set toolkit.enabled=false --set migManager.enabled=false`.
    * To work with _containerd_ (e.g. for Tanzu), use the [defaultRuntime](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#chart-customization-options){target=_blank} flag accordingly.
    * To use [Dynamic MIG](../../../Researcher/scheduling/fractions.md/#dynamic-mig), the GPU Operator must be installed with the flag `mig.strategy=mixed`. If the GPU Operator is already installed, edit the clusterPolicy by running ```kubectl patch clusterPolicy cluster-policy -n gpu-operator --type=merge -p '{"spec":{"mig":{"strategy": "mixed"}}}```
    * If you are using [DGX OS](https://docs.nvidia.com/dgx/index.html){target=_blank} then NVIDIA prerequisites are already installed and you may skip to the next step.
        

??? "Run:ai 2.3 or earlier"
    * Run:ai has customized the [NVIDIA device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin){target=_blank} and [NVIDIA DCGM Exporter](https://github.com/NVIDIA/gpu-monitoring-tools){target=_blank}. Run the following to disable the existing plug-ins:

    ```
    kubectl -n gpu-operator patch daemonset nvidia-device-plugin-daemonset \
    -p '{"spec": {"template": {"spec": {"nodeSelector": {"non-existing": "true"}}}}}'
    kubectl -n gpu-operator patch daemonset nvidia-dcgm-exporter \
    -p '{"spec": {"template": {"spec": {"nodeSelector": {"non-existing": "true"}}}}}'
    ```

    (use the namespace `gpu-operator-resources` if the NVIDIA GPU Operator is of version 1.8 or earlier )



### Operating System

Run:ai will work on any __Linux__ operating system that is supported by both Kubernetes and [NVIDIA](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html){target=_blank}. Having said that, Run:ai performs its internal tests on Ubuntu 20.04 (and CoreOS for OpenShift). The exception is GKE (Google Kubernetes Engine). At this point, Run:ai will only work with Ubuntu.

### Prometheus 

The Run:ai Cluster installation will, by default, install [Prometheus](https://prometheus.io/){target=_blank}, but it can also connect to an existing Prometheus instance installed by the organization. In the latter case, it's important to:

* Verify that both [Prometheus Node Exporter](https://prometheus.io/docs/guides/node-exporter/){target=_blank} and [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics){target=_blank} are installed. Both are part of the default Prometheus installation
* Understand how Prometheus has been installed. Whether [directly](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus) or with the [Prometheus Operator](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack). The distinction is important during the Run:ai Cluster installation.

### Inference

:octicons-versions-24: [Version 2.5](../../../home/whats-new-2022.md#may-2022-runai-version-25)

To use the Run:ai inference module you must pre-install [Knative Serving](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/){target=_blank}. Follow the instructions [here](https://knative.dev/docs/install/){target=_blank} to install. Run:ai is certified on Knative 1.4 and 1.5 with Kubernetes 1.22 and Knative 1.3 for Kubernetes 1.21.  

Post-install, you must configure Knative to use the Run:ai scheduler by running: 

```
kubectl patch configmap/config-features \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"kubernetes.podspec-schedulername":"enabled"}}'
```

#### Inference Autoscaling
Run:ai allows to autoscale a deployment according to various metrics:

1. GPU Utilization (%)
2. CPU Utilization (%)
3. Latency (milliseconds)
4. Throughput (requests/second)
5. Concurrency 
6. Any custom metric

Additional installation may be needed for some of the metrics as follows:

* Using _Throughput_ or _Concurrency_ does not require any additional installation.
* Any other metric will require installing the [HPA Autoscaler](https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/#install-optional-serving-extensions){target=_blank}.
* Using _GPU Utilization_, _Latency_ or _Custom metric_ will __also__ require the Prometheus adapter. The Prometheus adapter is part of the Run:ai installer and can be added by setting the `prometheus-adapter.enabled` flag to `true`. See [Customizing the Run:ai installation](./customize-cluster-install.md) for further information.

If you wish to use an _existing_ Prometheus adapter installation, you will need to configure it manually with the Run:ai Prometheus rules, specified in the Run:ai chart values under `prometheus-adapter.rules` field. See [Configure existing Prometheus Adapter](./prerequisites/configure-existing-prom-adapter.md)

### Distributed Training via Kubeflow MPI

Distributed training is the ability to run workloads on multiple nodes (not just multiple GPUs on the same node). Run:ai provides this capability via Kubeflow MPI. If you need this functionality, you will need to install the [Kubeflow MPI Operator](https://github.com/kubeflow/mpi-operator){target=_blank}. 


=== "Version 2.5 or later"
    Use the following [installation guide](https://github.com/kubeflow/mpi-operator#installation){target=_blank}. As per instruction:

    * Verify that the `mpijob` custom resource does not currently exist in the cluster by running `kubectl get crds | grep mpijobs`. If it does, delete it by running `kubectl delete crd mpijobs.kubeflow.org`
    * run `kubectl apply -f https://raw.githubusercontent.com/kubeflow/mpi-operator/master/deploy/v2beta1/mpi-operator.yaml`

=== "Version 2.4 or earlier"
    Use the following [installation guide](https://github.com/kubeflow/mpi-operator/tree/v0.2.3#installation){target=_blank}. As per instruction:

    * Clone tag `v0.2.3` (and not master)
    * `vi mpi-operator/deploy/v1alpha2/mpi-operator.yaml`
    * search for `mpioperator/mpi-operator:latest` and change it to `mpioperator/mpi-operator:v0.2.3`.
    * search for `mpioperator/kubectl-delivery:latest` and change it to `mpioperator/kubectl-delivery:v0.2.3`.


!!! Notes
    Kubeflow MPI requires containers to run as root, which will not work well when running on OpenShift or when PodSecurityPolicy is enabled in Kubernetes.
    

### Reporting on Workload Progress

The Run:ai [Researcher Library](../../../Researcher/researcher-library/rl-reporting.md) includes a _reporting_ module. The reporting module externalizes progress information about the run which is then available as a graph in the Job list. 

To enable externalizing the information you must install the __Prometheus Push Gateway__ as follows: 

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install runai-prometheus-pushgateway prometheus-community/prometheus-pushgateway \
    -n runai --set serviceMonitor.enabled=true --set serviceMonitor.namespace=runai
```

## Hardware Requirements

(see picture below)

*   (Production only) __Run:ai System__ Nodes: To reduce downtime and save CPU cycles on expensive GPU Machines, we recommend that production deployments will contain __two or more__ worker machines, designated for Run:ai Software. The nodes do not have to be dedicated to Run:ai, but for Run:ai purposes we would need:
    
    *   4 CPUs
    *   8GB of RAM
    *   50GB of Disk space  
    
*   __Shared data volume:__ Run:ai uses Kubernetes to abstract away the machine on which a container is running:

    * Researcher containers: The Researcher's containers need to be able to access data from any machine in a uniform way, to access training data and code as well as save checkpoints, weights, and other machine-learning-related artifacts. 
    * The Run:ai system needs to save data on a storage device that is not dependent on a specific node.  

    Typically, this is achieved via Network File Storage (NFS) or Network-attached storage (NAS).

* __Docker Registry:__ With Run:ai, Workloads are based on Docker images. For container images to run on any machine, these images must be downloaded from a docker registry rather than reside on the local machine (though this also is [possible](../../../researcher-setup/docker-to-runai/#image-repository)). You can use a public registry such as [docker hub](https://hub.docker.com/){target=_blank} or set up a local registry on-prem (preferably on a dedicated machine). Run:ai can assist with setting up the repository.

*  __Kubernetes:__ Production Kubernetes installation requires separate nodes for the Kubernetes master. For more details see your specific Kubernetes distribution documentation. 

![img/prerequisites.png](img/prerequisites.jpg)

## User requirements

__Usage of containers and images:__ The individual Researcher's work should be based on [container](https://www.docker.com/resources/what-container){target=_blank} images. 

## Network Requirements

__Internal networking:__ Kubernetes networking is an add-on rather than a core part of Kubernetes. Different add-ons have different network requirements. You should consult the documentation of the specific add-on on which ports to open. It is however important to note that unless special provisions are made, Kubernetes assumes __all__ cluster nodes can interconnect using __all__ ports. 

__Outbound network:__ Run:ai user interface runs from the cloud. All container nodes must be able to connect to the Run:ai cloud. Inbound connectivity (connecting from the cloud into nodes) is not required. If outbound connectivity is proxied/limited, the following exceptions should be applied: 

### During Installation

Run:ai requires an installation over the Kubernetes cluster. The installation access the web to download various images and registries. Some organizations place limitations on what you can pull from the internet. The following list shows the various solution components and their origin: 

<table border="1" style="width: 650px; margin-left: 0px; margin-right: auto;">
<tbody>
<tr>
<th scope="row" style="width: 114.375px;">Name</th>
<th scope="row" style="width: 308.92px;">Description</th>
<th scope="row" style="width: 227.102px;">URLs</th>
<th scope="row" style="width: 43.4659px;">Ports</th>
</tr>
<tr>
<td style="padding: 6px; width: 104.375px;">
<p>Run:ai  Repository</p>
</td>
<td style="padding: 6px; width: 298.92px;">
<p> The Run:ai Package Repository is hosted on Run:ai’s account on Google Cloud </p>
</td>
<td style="padding: 6px; width: 217.102px;">
<p> <a href="http://runai-charts.storage.googleapis.com/">runai-charts.storage.googleapis.com</a> </p>
</td>
<td style="padding: 6px; width: 33.4659px;">
<p>443</p>
</td>
</tr>
<tr>
<td style="padding: 6px; width: 104.375px;">
<p>Docker Images Repository</p>
</td>
<td style="padding: 6px; width: 298.92px;">
<p>Various Run:ai images</p>
</td>
<td style="padding: 6px; width: 217.102px;">
<p><a href="http://hub.docker.com/">hub.docker.com </a></p>
<p>gcr.io/run-ai-prod </p>
</td>
<td style="padding: 6px; width: 33.4659px;">
<p>443</p>
</td>
</tr>
<tr>
<td style="padding: 6px; width: 104.375px;">
<p> Docker Images Repository </p>
</td>
<td style="padding: 6px; width: 298.92px;">
<p> Various third party Images</p>
</td>
<td style="padding: 6px; width: 217.102px;">
<p><a href="http://quay.io/">quay.io</a>  </p>
</td>
<td style="padding: 6px; width: 33.4659px;">
<p>  443   </p>
</td>
</tr>

<tr>
<td style="padding: 6px; width: 106px;">
<p> Cert Manager </p>
</td>
<td style="padding: 6px; width: 304px;">
<p> Creates a letsencrypt-based certificate for the cluster </p>
</td>
<td style="padding: 6px; width: 205px;">
<p> 8.8.8.8, 1.1.1.1, dynu.com </p>
<p> </p>
</td>
<td style="padding: 6px; width: 32px;">
<p>53</p>
</td>

</tbody>
</table>

### Post Installation

In addition, once running, Run:ai requires outbound network connection to the following targets:

<table border="1" style="margin-left: 0px; margin-right: auto; width: 650px;">
<tbody>
<tr style="height: 22px;">
<th scope="row" style="width: 116px; height: 22px;">Name</th>
<th scope="row" style="width: 314px; height: 22px;">Description</th>
<th scope="row" style="width: 215px; height: 22px;">URLs</th>
<th scope="row" style="width: 42px; height: 22px;">Ports</th>
</tr>
<tr>
<td style="padding: 6px; width: 106px;">
<p>Grafana</p>
</td>
<td style="padding: 6px; width: 304px;">
<p>Grafana Metrics Server</p>
</td>
<td style="padding: 6px; width: 205px;">
<p>prometheus-us-central1.grafana.net</p>
</td>
<td style="padding: 6px; width: 32px;">
<p>443 </p>
</td>
</tr>
<tr>
<td style="padding: 6px; width: 106px;">
<p> Run:ai </p>
</td>
<td style="padding: 6px; width: 304px;">
<p> Run:ai   Cloud instance </p>
</td>
<td style="padding: 6px; width: 205px;">
<p> <a href="https://app.run.ai">app.run.ai</a> </p>
<p> </p>
</td>
<td style="padding: 6px; width: 32px;">
<p>443</p>
</td>
<tr>
<td style="padding: 6px; width: 106px;">
<p> Auth0 </p>
</td>
<td style="padding: 6px; width: 304px;">
<p> Authentication Provider (older tenants only) </p>
</td>
<td style="padding: 6px; width: 205px;">
<p> <a href="https://runai-prod.auth0.com/">runai-prod.auth0.com</a> </p>
<p> </p>
</td>
<td style="padding: 6px; width: 32px;">
<p>443</p>
</td>

<tr>
<td style="padding: 6px; width: 106px;">
<p> Cert Manager </p>
</td>
<td style="padding: 6px; width: 304px;">
<p> Creates a letsencrypt-based certificate for the cluster </p>
</td>
<td style="padding: 6px; width: 205px;">
<p> 8.8.8.8, 1.1.1.1, dynu.com </p>
<p> </p>
</td>
<td style="padding: 6px; width: 32px;">
<p>53</p>
</td>

</tbody>
</table>

### Cluster IP

The Run:ai user interface requires an IP address to the Kubernetes cluster. The requirement is relevant for SaaS installation only. Following are instructions on how to get the IP and set firewall settings. 


=== "Unmanaged Kubernetes" 
    * Use the node IP of any of the Kubernetes nodes. 
    * Setup the firewall such that the IP is available to Researchers running within the organization (but not outside the organization).

=== "Unmanaged Kubernetes on the cloud" 
    * Use the node IPs of any of the Kubernetes nodes. Both internal and external IP in the format __external-IP,internal-IP__. 
    * Setup the firewall such that the external IP is available to Researchers running within the organization (but not outside the organization).


=== "Managed Kubernetes (EKS, AKS, GKE)"

    You will need to externalize an IP address via a load balancer. If you do not have an existing load balancer already, install __NGINX__ as follows:

    ``` bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install nginx-ingress ingress-nginx/ingress-nginx
    ```

    Find the Cluster IP by running:

    ``` bash
    echo $(kubectl get svc nginx-ingress-ingress-nginx-controller  -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
    ```


## Pre-install Script

Once you believe that the Run:ai prerequisites are met, we highly recommend installing and running the Run:ai  [pre-install diagnostics script](https://github.com/run-ai/preinstall-diagnostics){target=_blank}. The tool:

* Tests the below requirements as well as additional failure points related to Kubernetes, NVIDIA, storage, and networking.
* Looks at additional components installed and analyzes their relevancy to a successful Run:ai installation. 

To use the script [download](https://github.com/run-ai/preinstall-diagnostics/releases){target=_blank} the latest version of the script and run:

```
chmod +x preinstall-diagnostics-<platform>
./preinstall-diagnostics-<platform>
```

If the script fails, or if the script succeeds but the Kubernetes system contains components other than Run:ai, locate the file `runai-preinstall-diagnostics.txt` in the current directory and send it to Run:ai technical support. 

For more information on the script including additional command-line flags, see [here](https://github.com/run-ai/preinstall-diagnostics){target=_blank}.

