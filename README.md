# proxmox-terraform-k8s
 
Template for setting up VMs for a k8s cluster on a Proxmox node.

Also creates a handy ansible inventory in ansible/hosts.

This repo is using the telmate/proxmox provider for terraform.

Make sure to read the [documentation](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs) to understand all the variables being used in the variables.tf file

## Important variables to update

All mandatory variables are put in a file named 'terraform.tfvars'.
You can make this file by copying terraform.tfvars.example and updating the values in it.
```
cp terraform.tfvars.example terraform.tfvars
# Edit this file and save it
vim terraform.tfvars
```
Description for all vars in terraform.tfvars is available in 'variables.tf' file
Apart from the variables mentioned above you can also edit other variables in 'variables.tf' file.

The variables mentioned below are hard-coded in main.tf as I don't think most people would move away from these defaults. Please update main.tf if you need other defaults.

```
<!-- Proxmox TLS check -->
pm_tls_insecure = true
<!-- Full cloning for all VMs -->
full_clone = true
<!-- This one is explained below in 'Extra configs' section -->
guest_agent_ready_timeout = 60
<!-- SSH port for all VMs -->
ansible_port = 22
```

**CLONE_TEMPLATE should be configured before creating the VMs**

Learn more about how to create a template [here](https://pve.proxmox.com/wiki/VM_Templates_and_Clones#Create_VM_Template)
```
ssh to Proxmox and download ubuntu iso
root@pve:/var/lib/vz/template/iso# wget http://releases.ubuntu.com/21.04/ubuntu-21.04-live-server-amd64.iso

CREATE VM: ubuntu20.04 ---> convert VM to template (template name 'ubuntu') Ref://support.us.ovhcloud.com/hc/en-us/articles/360010916620-How-to-Create-a-VM-in-Proxmox-VE

```
## Extra configs
There are some params that are not mentioned in the docs. Look [here](https://github.com/Telmate/terraform-provider-proxmox/blob/master/proxmox/resource_vm_qemu.go) to learn more.

One of the params that we're using is 'guest_agent_ready_timeout', its been set to '60' as it speeds up the creation of VMs this way. Look at this [issue](https://github.com/Telmate/terraform-provider-proxmox/issues/325) to learn more.

## Create the VMs
```
terraform init
terraform plan
terraform apply
```

<img src="pictures/proxmox-k8s.png?raw=true" width="900">

This will also create an ansible inventory file. You can check if its formatted correctly by
```
ansible-inventory -v --list -i ansible/hosts
```

You can update 'hosts.tmpl' if you prefer some other format for your ansible inventory.

Example:

```
$ terraform apply 
...

proxmox_vm_qemu.kworker[0]: Creating...
proxmox_vm_qemu.kmaster[0]: Creating...
proxmox_vm_qemu.kworker[1]: Creating...

$ ssh davar@192.168.1.22 -o IdentitiesOnly=yes
# hostnamectl set-hostname kmaster1
# curl -sfL https://get.k3s.io | sh -

Get K3S_TOKEN (stored at /var/lib/rancher/k3s/server/node-token on your k8s master server node).

To install on worker nodes and add them to the cluster, run the installation script with the K3S_URL and K3S_TOKEN environment variables. Here is an example showing how to join a worker node

Note: Each machine must have a unique hostname.

$ ssh davar@192.168.1.20 -o IdentitiesOnly=yes
# hostnamectl set-hostname kworker0
# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.22:6443 K3S_TOKEN="K10073946490225175ee91bc5b1a042bf90c0a64e20cc250c6e47c1b8dbaf4ba4f9::server:eb7864f8e04971d7be27299f340fdeaf" sh -

$ ssh davar@192.168.1.21 -o IdentitiesOnly=yes
# hostnamectl set-hostname kworker1
# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.22:6443 K3S_TOKEN="K10073946490225175ee91bc5b1a042bf90c0a64e20cc250c6e47c1b8dbaf4ba4f9::server:eb7864f8e04971d7be27299f340fdeaf" sh -

root@kmaster0:~# kubectl get node
NAME       STATUS   ROLES                  AGE    VERSION
ubuntu     Ready    control-plane,master   32m    v1.21.5+k3s1
kworker0   Ready    <none>                 105s   v1.21.5+k3s1
kworker1   Ready    <none>                 70s    v1.21.5+k3s1
root@kmaster0:~# kubectl get all --all-namespaces
NAMESPACE     NAME                                          READY   STATUS      RESTARTS   AGE
kube-system   pod/metrics-server-86cbb8457f-zxzxn           1/1     Running     0          33m
kube-system   pod/local-path-provisioner-5ff76fc89d-kdlfv   1/1     Running     0          33m
kube-system   pod/helm-install-traefik-crd-g29jv            0/1     Completed   0          33m
kube-system   pod/helm-install-traefik-9bhzr                0/1     Completed   1          33m
kube-system   pod/svclb-traefik-jhlbb                       2/2     Running     0          32m
kube-system   pod/coredns-7448499f4d-r478w                  1/1     Running     0          33m
kube-system   pod/traefik-97b44b794-jhkfx                   1/1     Running     0          32m
kube-system   pod/svclb-traefik-b88v2                       2/2     Running     0          2m19s
kube-system   pod/svclb-traefik-d6rbw                       2/2     Running     0          103s

NAMESPACE     NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP                              PORT(S)                      AGE
default       service/kubernetes       ClusterIP      10.43.0.1       <none>                                   443/TCP                      33m
kube-system   service/kube-dns         ClusterIP      10.43.0.10      <none>                                   53/UDP,53/TCP,9153/TCP       33m
kube-system   service/metrics-server   ClusterIP      10.43.174.122   <none>                                   443/TCP                      33m
kube-system   service/traefik          LoadBalancer   10.43.132.168   192.168.1.20,192.168.1.21,192.168.1.22   80:31586/TCP,443:30156/TCP   32m

NAMESPACE     NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/svclb-traefik   3         3         3       3            3           <none>          32m

NAMESPACE     NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/metrics-server           1/1     1            1           33m
kube-system   deployment.apps/coredns                  1/1     1            1           33m
kube-system   deployment.apps/local-path-provisioner   1/1     1            1           33m
kube-system   deployment.apps/traefik                  1/1     1            1           32m

NAMESPACE     NAME                                                DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/metrics-server-86cbb8457f           1         1         1       33m
kube-system   replicaset.apps/coredns-7448499f4d                  1         1         1       33m
kube-system   replicaset.apps/local-path-provisioner-5ff76fc89d   1         1         1       33m
kube-system   replicaset.apps/traefik-97b44b794                   1         1         1       32m

NAMESPACE     NAME                                 COMPLETIONS   DURATION   AGE
kube-system   job.batch/helm-install-traefik-crd   1/1           42s        33m
kube-system   job.batch/helm-install-traefik       1/1           46s        33m

Clean:
$ terraform destroy 
```
