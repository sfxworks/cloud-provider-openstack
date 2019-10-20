# Using with kubeadm

## Prerequisites

- kubeadm, kubelet and kubectl has been installed.

## Steps

- Create your kubeadm config file

    ```
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        cloud-provider: "external"
    ---
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterConfiguration
    kubernetesVersion: stable
    networking:
      podSubnet: 10.244.0.0/16 
    controllerManager:
      extraArgs:
        "external-cloud-volume-plugin": "openstack"
      extraVolumes:
      - name: "cloud-config"
        hostPath: "/etc/kubernetes/cloud-config"
        mountPath: "/etc/kubernetes/cloud-config"
        readOnly: true
        pathType: FileOrCreate
    ```

- Create the cloud config file `/etc/kubernetes/cloud-config` on each node. You can find an example file in `manifests/controller-manager/cloud-config`.

    > `/etc/kubernetes/cloud-config` is the default cloud config file path used by controller-manager and kubelet.

- Bootstrap the cluster on the master node.

    ```
    kubeadm init --config kubeadm.conf
    ```

    You can find an example kubeadm.conf in `manifests/controller-manager/kubeadm.conf`. Follow the usual steps to install the network plugin and then bootstrap the other nodes using `kubeadm join`. The provided configuration 

- Create a secret containing the cloud configuration for cloud-controller-manager.

   Update `cloud.conf` configuration in `manifests/controller-manager/cloud-config-secret.yaml`:

    ```shell
    cp /etc/kubernetes/cloud.conf ~/cloud.conf
    kubectl create secret generic -n kube-system cloud-config --from-file=~/cloud.conf
    ```

- Before we create cloud-controller-manager deamonset, you can find all the nodes have the taint `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule` and waiting for being initialized by cloud-controller-manager.

- Create RBAC resources and cloud-controller-manager deamonset.

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/cluster/addons/rbac/cloud-controller-manager-roles.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/cluster/addons/rbac/cloud-controller-manager-role-bindings.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/cloud-provider-openstack/master/manifests/controller-manager/openstack-cloud-controller-manager-ds.yaml
    ```

- After the cloud-controller-manager deamonset is up and running, the node taint above will be removed by cloud-controller-manager, you can also see some more information in the node label.


## Where to go next: 
- Enable Block Storage using the [Cinder Container Storage Interface](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-cinder-csi-plugin.md)
- Encrypt Secrets at rest using the [Barbican KMS Plugin](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-barbican-kms-plugin.md)
- Expose applications with service type [Load Balancer](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/expose-applications-using-loadbalancer-type-service.md)
- Route traffic using the [Octavia Ingress Controller](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-octavia-ingress-controller.md) 

_Optionally consider kayrus' [Terraform Ingress Controller](https://github.com/kayrus/ingress-terraform) which supports TLS Termination. Note: currently in alpha_
