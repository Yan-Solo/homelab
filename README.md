# How to set up my home ops

## Required tooling

* Hypervisor with [Proxmox](https://www.proxmox.com/en/) installed, configured and with the latest [Talos ISO](https://www.talos.dev/v1.0/introduction/getting-started/#acquire-the-installation-image) uploaded
* [talosctl](https://www.talos.dev/v1.0/introduction/getting-started/#prerequisites) on your workstation
* [kubectl](https://kubernetes.io/docs/tasks/tools/) on your workstation
* [flux](https://fluxcd.io/docs/installation/) on your workstation
* [helm](https://helm.sh/docs/intro/install/) on your workstation (optional)

# Set up VMs

Create the amount of master and worker nodes you want to use with the latest Talos ISO image in Proxmox. Due to my limitations in hardware I only have 1 master and 2 workers. This is enough for everything I want to deploy, however this does not represent what a production ready high available cluster would look like.

## Configure the VMs to run a Kubernetes cluster with Talos


:::info
I might automate this process in the future, Sidero is said to be working on a Terraform provider for Talos

:::

Look at the console of all the VMs and note the IP it got from the DHCP server. If you run your own DHCP server refer to the Talos documentation. I don't do this because it's too much hassle with my ISP and their provided router. **Insert screenshot from proxmox console to look at the IP** As you can see the IP for the master node is 192.168.0.X in the screenshot above.

In this specific scenario I have one master and two worker nodes. So this README will be written to accommodate that. It will also assume you are executing command from the root directory of this repo. And for the sake of this example, the table below will contain the IPs that you could get from your DHCP server.

| Node | IP |
|----|----|
| talos-master-01 | 192.168.0.138 |
| talos-worker-01 | 192.168.0.238 |
| talos-worker-02 | 192.168.0.223 |

The next table will show what node will have what static IP each node will have after applying the configuration

| Node | IP |
|----|----|
| talos-master-01 | 192.168.0.80 |
| talos-worker-01 | 192.168.0.81 |
| talos-worker-02 | 192.168.0.82 |

It might also be useful for you to edit some of the config in the talosconfig folder such as NTP server. Or if you want other static IP addresses. For a full list of what is configurable look at the [Talos docs](https://www.talos.dev/v1.0/reference/configuration/)

Now, let's get into configuring the nodes. It only involves a couple of commands. This is more or less the same as the [docs](https://www.talos.dev/v1.0/talos-guides/install/virtualized-platforms/proxmox/#create-control-plane-node).

You can watch each nodes console if you like to know if something is actually happening.

### Master-01

```bash
$ talosctl apply-config --insecure --nodes 192.168.0.138 --file talosconfig/master-01.yaml
```

### Worker-01

```bash
$ talosctl apply-config --insecure --nodes 192.168.0.238 --file talosconfig/worker-01.yaml
```

### Worker-02

```bash
$ talosctl apply-config --insecure --nodes 192.168.0.223 --file talosconfig/worker-02.yaml
```

### Bootstrap the cluster

```bash
$ export TALOSCONFIG="talosconfig/talosconfig"
$ export MASTER_IP="192.168.0.80"

$ talosctl config endpoint $MASTER_IP
$ talosctl config node $MASTER_IP
$ talosctl bootstrap
```

## Retrieve your kubeconfig

At this point you can get your kubeconfig with a single command and place it in the directory of this repo.

```bash
$ export TALOSCONFIG="talosconfig/talosconfig"
$ talosctl kubeconfig .
```


Now you can test the cluster and the config

```bash
$ export KUBECONFIG="kubeconfig"
$ kubectl get nodes
NAME              STATUS   ROLES                  AGE   VERSION
talos-master-01   Ready    control-plane,master   43m   v1.23.6
talos-worker-01   Ready    <none>                 42m   v1.23.6
talos-worker-02   Ready    <none>                 42m   v1.23.6
```

# Bootstrap flux


:::tip
This section is to bootstrap **this** flux config onto a new cluster and apply my manifests into the cluster. If you want to learn how to bootstrap a *clean* flux into a cluster you can take a look at their [documentation](https://fluxcd.io/docs/installation/#bootstrap)

:::


