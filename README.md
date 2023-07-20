# A Kubernetes Cloud Shell (Web Terminal) Operator

[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/6301/badge)](https://bestpractices.coreinfrastructure.org/projects/6301)

English | [简体中文](https://github.com/cloudtty/cloudtty/blob/main/README_zh.md)

cloudtty is an easy-to-use operator to run web terminal and cloud shell intended for a kubernetes-native environment.
You can easily open a terminal on your own browser via cloudtty.
The community is always open for any contributor and those who want to have a try.

Literally, cloudtty herein refers to a virtual console, shell, or terminal running on web and clouds.
You can use it anywhere with an internet conneciton and it will be automatically connected to the cloud.

> Early user terminals connected to computers were electromechanical teleprinters or teletypewriters (TeleTYpewriter, TTY), which might be the origin of TTY.
> Gradually, TTY has continued to be used as the name for a text-only console although now this text-only console is a virtual console not a physical console.

<img src="https://github.com/cncf/artwork/blob/master/other/illustrations/ashley-mcnamara/transparent/cncf-cloud-gophers-transparent.png" style="width:600px;" />

**cloudtty is a [Cloud Native Computing Foundation](https://cncf.io/) landscape project.**

## Why Do You Need cloudtty?

A project [ttyd](https://github.com/tsl0922/ttyd) provides some features to share terminals over the web.
But if you use Kubernetes, a more cloud-native enviroment is required to run the webtty via the `kubernetes` way (running as a pod, and generated by CRDs),
which is covered by cloudtty. You are welcome to try cloudtty :tada:.

## Applicable Scenarios

1. Many enterprises use a cloud platform to manage Kubernetes, but due to security reasons,
   you cannot use SSH to connect the node host to execute `kubectl` commands.
   In this case, you may require a cloud shell capability.

2. A running container on kubernetes can be "entered" (via `Kubectl exec`) on a browser web page.

3. The container logs can be displayed in real time (scrolling) on a browser web page.

## Demo

![screenshot_gif](https://github.com/cloudtty/cloudtty/raw/main/docs/snapshot.gif)

After the cloudtty is intergated to your own UI, it would look like:

![demo_png](https://github.com/cloudtty/cloudtty/raw/main/docs/demo.png)

## Quick Start

- Step 1: Install the Operator and CRDs

  a. Install the operator using Helm

  ```shell
  helm repo add cloudtty https://cloudtty.github.io/cloudtty
  helm repo update
  helm install cloudtty-operator --version 0.5.0 cloudtty/cloudtty
  ```

  b. Wait for the operator pod until it is running

  ```shell
  kubectl wait deployment cloudtty-operator-controller-manager --for=condition=Available=True
  ```

- Step 2: Create a cloudtty instance by applying CR, and then monitor its status

  ```shell
  kubectl apply -f https://raw.githubusercontent.com/cloudtty/cloudtty/v0.5.0/config/samples/local_cluster_v1alpha1_cloudshell.yaml
  ```

  By default, it will create a cloudtty pod and expose the `NodePort` service.
  Alternatively, `Cluster-IP`, `Ingress`, and `Virtual Service`(for Istio) are all supported as `exposureMode`, please refer to `config/samples/` for more examples.

- Step 3: Observe CR status to obtain its web access url, such as:

  ```shell
  kubectl get cloudshell -w
  ```

  You can see the following information:

  ```shell
  NAME                 USER   COMMAND  TYPE        URL                 PHASE   AGE
  cloudshell-sample    root   bash     NodePort    192.168.4.1:30167   Ready   31s
  cloudshell-sample2   root   bash     NodePort    192.168.4.1:30385   Ready   9s
  ```

  When the status of cloudshell changes to `Ready` and the `URL` field appears, copy and paste the URL on your browser to access the cluster with `kubectl`, as shown below:

  ![screenshot_png](https://github.com/cloudtty/cloudtty/raw/main/docs/snapshot.png)

### How to build customize cloudshell image

Most users need more than just the basic `kubectl` tools to manage their clusters. we can customize image based on cloudshell base image. here is an example of adding the `karmadactl` tool.

- Modify [Dockerfile.example](https://github.com/cloudtty/cloudtty/blob/main/docker/Dockerfile.example).

  ```shell
  FROM ghcr.io/cloudtty/cloudshell:v0.5.0

  RUN curl -fsSLO https://github.com/karmada-io/karmada/releases/download/v1.2.0/kubectl-karmada-linux-amd64.tgz \
      && tar -zxf kubectl-karmada-linux-amd64.tgz \
      && chmod +x kubectl-karmada \
      && mv kubectl-karmada /usr/local/bin/kubectl-karmada \
      && which kubectl-karmada

  ENTRYPOINT ttyd
  ```

- Rebuild new image with `karmadactl` tool：

  ```shell
  docker build -t <IMAGE> . -f docker/Dockerfile-webtty
  ```

### Use customize cloudshell image

There are two ways to set the customized cloudshell image:

1. You can set the image directly by using the cloudshell CR field `spec.image`.

    ```yaml
    apiVersion: cloudshell.cloudtty.io/v1alpha1
    kind: CloudShell
    metadata:
      name: cloudshell-sample
    spec:
      secretRef: 
        name: "my-kubeconfig"
      image: ghcr.io/cloudtty/customize_cloudshell:latest
    ```

2. Set the 'JobTemplate' image parameter to run the customized cloudshell image when installing cloudtty.

    ```shell
    helm install cloudtty-operator --version 0.5.0 cloudtty/cloudtty --set jobTemplate.image.registry=</REGISTRY> --set jobTemplate.image.repository=</REPOSITORY> --set jobTemplate.image.tag=</TAG>
    ```

> If you have installed cloudtty, you can also modify the configMap of JobTemplate to set the cloudshell image.

## Advanced Usage Guide

### Manage Multiple or Remote Clusters

If cloudtty manages a remote cluster (another cluster than which the cloudtty operator runs on),
you need tell cloudtty the kube.conf of the remote cluster as below.

You can copy the kube.config, `~/.kube/config`, from a remote cluster.

```shell
kubectl create secret generic my-kubeconfig --from-file=kube.config
```

Be careful to ensure the `/root/.kube/config`:

1. contains the base64 encoded certs/secrets instead of local files.

2. can reach the k8s api-server endpoint (via host IP or cluster IP) instead of localhost.

- If the cluster is remote, `cloudtty` needs to specify `kubeconfig` to access the cluster using the kubectl command tool.
  You need to provide the kubeconfig stored in secret and specify the name to cloudshell `spec.secretRef.name` CR.
  kubeconfig will be automatically mounted to the cloudtty container.
  Ensure that the server IP address is properly connected to the cluster network.

- If cloudtty runs on the same cluster which to be managed,
  you don't need to do this (a ServiceAccount with `cluster-admin` role permissions will be binded to the pod automaticlly.
  Inside the container, kubectl automatically detects `CA` certificates and token.
  If any concern with security, you can also provide your own kubeconfig to control the permissions for different users.)

### Manager cluster node

The basic image to cloudshell had integrated with the plugin of [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell), you can use its command to connect an arbitrary node of specified cluster. It will run a pod with privilege, if you attach importance to the pod security, please be careful with the feature. See the following sample:

```yaml
apiVersion: cloudshell.cloudtty.io/v1alpha1
kind: CloudShell
metadata:
  name: cloudshell-node-shell
spec:
  secretRef:
    name: "<KUBECONFIG>"
  commandAction: "kubectl node-shell <NODE_NAME>"
```

For more samples refer to [kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell).

> If a cluster has the security policy such as `PodSecurity` and `PSP`, the feature may be affected.

### More Exposure Modes

Cloudtty provides the following four modes to expose cloudtty services to satisfy different usage scenarios:

- `ClusterIP`: [Service](https://kubernetes.io/docs/concepts/services-networking/service/) ClusterIP type in a cluster,
  which is suitable for third-party integration of cloudtty server. You can choose a more flexible way to expose your services.

- `NodePort` (default): The simplest way to expose the service mode is to create a service resource with type NodePort in a cluster.
  You can access the cloudtty service using the master node IP address and port number.

- `Ingress`: Create a Service resource of ClusterIP type in a cluster and create an ingress resource to load the service based on routing rules.
  This works when the [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) is used for traffic load in the cluster.

- `VirtualService` (Istio): Create a ClusterIP Service resource in a cluster and create a `VirtaulService` resource.
  This mode is used when [Istio](https://github.com/istio/istio) is used to load traffic in a cluster.

### featureGate

`AllowSecretStoreKubeconfig`：Restore kubeconfig file with secret resource, if the featureGate is enabled, the field `spec.configmapName` will be disabled.
You can use the field `spec.secretRef.name` to difine where kubeconfig is. Currently the featureGate is in alpha phase, disabled by default.

#### How to open featrueGate

1. If you use `yaml` to deploy cloudtty, add `--feature-gates=AllowSecretStoreKubeconfig=true` to operator to run arguments.
2. If you use `helm` to deploy cloudtty, set the parameter `--set image.featureGates.AllowSecretStoreKubeconfig=true`.

## Rationale

1. Operator creates a job and a service with the same name in the proper namespace.
   If Ingress or VitualService is used, it also creates the routing information.

2. When the pod status is `Ready`, it will show the access url to the cloudshell status.

3. When a [job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) ends after the TTL is expired or the job is terminated for some other reasons,
   the cloudshell status changes to `Completed` once the job changes to `Completed`.
   You can set cloudshell to delete associated resources when the status is `Completed`.

4. When cloudshell is deleted, the corresponding job and service (through 'ownerReference') are automatically deleted.
   If Ingress or VitualService mode is used, the corresponding routing information will be deleted too.

## Developer Guide

### Run the Operator and Install CRDs

1. Generate CRDs to [charts/_crds](charts/cloudtty/_crds/)

    ```shell
    make generate-yaml
    ```

2. Install CRDs

    ```shell
    make install
    ```

3. Run the operator

    ```shell
    make run
    ```

### Create cloudshell

For example, automatically print logs for a container.

```yaml
apiVersion: cloudshell.cloudtty.io/v1alpha1
kind: CloudShell
metadata:
  name: cloudshell-sample
spec:
  secretRef:
    name: "my-kubeconfig"
  runAsUser: "root"
  commandAction: "kubectl -n kube-system logs -f kube-apiserver-cn-stack"
  once: false
```

## Special Thanks

This project is based on https://github.com/tsl0922/ttyd. Many thanks to `tsl0922`, `yudai`, and the community.
The frontend UI code was originated from `ttyd` project, and the ttyd binary inside the container also comes from `ttyd` project.

## Discussion

If you have any question, feel free to reach out to us in the following ways:

- [Slack Channel](https://cloud-native.slack.com/archives/C03LA6AUF7V)

- WeChat Group: contact `calvin0327`(wen.chen@daocloud.io) to join

## What's Next

- Control permissions through RBAC (to generate the `/var/run/secret` file).

- For security, jobs should run in separate namespaces, not in the namespace same as  cloudshell.

- Check the pod is running and endpoint status changes to `Ready`, and the cloudshell phase is set to `Ready`.

- TTL should be set to both job and shell.

- Job creation templates are currently hardcode and should provide a more flexible way to modify the job template.

More will be coming Soon. Welcome to [open an issue](https://github.com/cloudtty/cloudtty/issues) and [propose a PR](https://github.com/cloudtty/cloudtty/pulls). 🎉🎉🎉

## Contributors

<a href="https://github.com/cloudtty/cloudtty/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=cloudtty/cloudtty" />
</a>

Made with [contrib.rocks](https://contrib.rocks).

<p align="center">
<img src="https://landscape.cncf.io/images/left-logo.svg" width="300"/>&nbsp;&nbsp;<img src="https://landscape.cncf.io/images/right-logo.svg" width="350"/>
<br/><br/>
cloudtty enriches the <a href="https://landscape.cncf.io/?selected=cloud-tty">CNCF CLOUD NATIVE Landscape.</a>
</p>
