# Digital Ocean

In this tutorial, we'll create a Kubernetes v1.8.1 cluster on Digital Ocean.

We'll declare a Kubernetes cluster in Terraform using the Typhoon Terraform module. On apply, firewall rules, DNS records, tags, and droplets for Kubernetes controllers and workers will be created.

Controllers and workers are provisioned to run a `kubelet`. A one-time [bootkube](https://github.com/kubernetes-incubator/bootkube) bootstrap schedules an `apiserver`, `scheduler`, `controller-manager`, and `kube-dns` on controllers and runs `kube-proxy` and `flannel` on each node. A generated `kubeconfig` provides `kubectl` access to the cluster.

## Requirements

* Digital Ocean Account and Token
* Digital Ocean Domain (registered Domain Name or delegated subdomain)
* Terraform v0.10.1+ and [terraform-provider-ct](https://github.com/coreos/terraform-provider-ct) installed locally

## Terraform Setup

Install [Terraform](https://www.terraform.io/downloads.html) v0.10.1+ on your system.

```sh
$ terraform version
Terraform v0.10.1
```

Add the [terraform-provider-ct](https://github.com/coreos/terraform-provider-ct) plugin binary for your system.

```sh
wget https://github.com/coreos/terraform-provider-ct/releases/download/v0.2.0/terraform-provider-ct-v0.2.0-linux-amd64.tar.gz
tar xzf terraform-provider-ct-v0.2.0-linux-amd64.tar.gz
sudo mv terraform-provider-ct-v0.2.0-linux-amd64/terraform-provider-ct /usr/local/bin/
```

Add the plugin to your `~/.terraformrc`.

```
providers {
  ct = "/usr/local/bin/terraform-provider-ct"
}
```

Read [concepts](concepts.md) to learn about Terraform, modules, and organizing resources. Change to your infrastructure repository (e.g. `infra`).

```
cd infra/clusters
```

## Provider

Login to [DigitalOcean](https://cloud.digitalocean.com) or create an [account](https://cloud.digitalocean.com/registrations/new), if you don't have one.

Generate a Personal Access Token with read/write scope from the [API tab](https://cloud.digitalocean.com/settings/api/tokens). Write the token to a file that can be referenced in configs.

```sh
mkdir -p ~/.config/digital-ocean
echo "TOKEN" > ~/.config/digital-ocean/token
```

Configure the DigitalOcean provider to use your token in a `providers.tf` file.

```tf
provider "digitalocean" {
  token = "${chomp(file("~/.config/digital-ocean/token"))}"
}
```

## Cluster

Define a Kubernetes cluster using the module `digital-ocean/container-linux/kubernetes`.

```tf
module "digital-ocean-nemo" {
  source = "git::https://github.com/poseidon/typhoon//digital-ocean/container-linux/kubernetes"

  region   = "nyc3"
  dns_zone = "digital-ocean.example.com"

  cluster_name     = "nemo"
  image            = "coreos-stable"
  controller_count = 1
  controller_type  = "2gb"
  worker_count     = 2
  worker_type      = "512mb"
  ssh_fingerprints = ["d7:9d:79:ae:56:32:73:79:95:88:e3:a2:ab:5d:45:e7"]

  # output assets dir
  asset_dir = "/home/user/.secrets/clusters/nemo"
}
```

Reference the [variables docs](#variables) or the [variables.tf](https://github.com/poseidon/typhoon/blob/master/digital-ocean/container-linux/kubernetes/variables.tf) source.

## ssh-agent

Initial bootstrapping requires `bootkube.service` be started on one controller node. Terraform uses `ssh-agent` to automate this step. Add your SSH private key to `ssh-agent`.

```sh
ssh-add ~/.ssh/id_rsa
ssh-add -L
```

!!! warning
    `terrafrom apply` will hang connecting to a controller if `ssh-agent` does not contain the SSH key.

## Apply

Initialize the config directory if this is the first use with Terraform.

```sh
terraform init
```

Get or update Terraform modules.

```sh
$ terraform get            # downloads missing modules
$ terraform get --update   # updates all modules
Get: git::https://github.com/poseidon/typhoon (update)
Get: git::https://github.com/poseidon/bootkube-terraform.git?ref=v0.8.0 (update)
```

Plan the resources to be created.

```sh
$ terraform plan
Plan: 54 to add, 0 to change, 0 to destroy.
```

Apply the changes to create the cluster.

```sh
$ terraform apply
module.digital-ocean-nemo.null_resource.bootkube-start: Still creating... (30s elapsed)
module.digital-ocean-nemo.null_resource.bootkube-start: Provisioning with 'remote-exec'...
...
module.digital-ocean-nemo.null_resource.bootkube-start: Still creating... (6m20s elapsed)
module.digital-ocean-nemo.null_resource.bootkube-start: Creation complete (ID: 7599298447329218468)

Apply complete! Resources: 54 added, 0 changed, 0 destroyed.
```

In 3-6 minutes, the Kubernetes cluster will be ready.

## Verify

[Install kubectl](https://coreos.com/kubernetes/docs/latest/configure-kubectl.html) on your system. Use the generated `kubeconfig` credentials to access the Kubernetes cluster and list nodes.

```
$ KUBECONFIG=/home/user/.secrets/clusters/nemo/auth/kubeconfig
$ kubectl get nodes
NAME             STATUS    AGE       VERSION
10.132.110.130   Ready     10m       v1.8.1+coreos.0
10.132.115.81    Ready     10m       v1.8.1+coreos.0
10.132.124.107   Ready     10m       v1.8.1+coreos.0
```

List the pods.

```
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
kube-system   kube-apiserver-n10qr                       1/1       Running   0          11m
kube-system   kube-controller-manager-3271970485-37gtw   1/1       Running   1          11m
kube-system   kube-controller-manager-3271970485-p52t5   1/1       Running   0          11m
kube-system   kube-dns-1187388186-ld1j7                  3/3       Running   0          11m
kube-system   kube-flannel-1cq1v                         2/2       Running   0          11m
kube-system   kube-flannel-hq9t0                         2/2       Running   1          11m
kube-system   kube-flannel-v0g9w                         2/2       Running   0          11m
kube-system   kube-proxy-6kxjf                           1/1       Running   0          11m
kube-system   kube-proxy-fh3td                           1/1       Running   0          11m
kube-system   kube-proxy-k35rc                           1/1       Running   0          11m
kube-system   kube-scheduler-3895335239-2bc4c            1/1       Running   0          11m
kube-system   kube-scheduler-3895335239-b7q47            1/1       Running   1          11m
kube-system   pod-checkpointer-pr1lq                     1/1       Running   0          11m
kube-system   pod-checkpointer-pr1lq-10.132.115.81       1/1       Running   0          10m
```

## Going Further

Learn about [version pinning](concepts.md#versioning), maintenance, and [addons](addons/overview.md).

!!! note
    On Container Linux clusters, install the `container-linux-update-operator` addon to coordinate reboots and drains when nodes auto-update. Otherwise, updates may not be applied until the next reboot.

## Variables

### Required

| Name | Description | Example |
|:-----|:------------|:--------|
| cluster_name | Unique cluster name (prepended to dns_zone) | nemo |
| region | Digital Ocean region | nyc1, sfo2, fra1, tor1 |
| dns_zone | Digital Ocean domain (i.e. DNS zone) | do.example.com |
| ssh_fingerprints | SSH public key fingerprints | ["d7:9d..."] |
| asset_dir | Path to a directory where generated assets should be placed (contains secrets) | /home/user/.secrets/nemo |

#### DNS Zone

Clusters create DNS A records `${cluster_name}.${dns_zone}` to resolve to controller droplets (round robin). This FQDN is used by workers and `kubectl` to access the apiserver. In this example, the cluster's apiserver would be accessible at `nemo.do.example.com`.

You'll need a registered domain name or subdomain registered in Digital Ocean Domains (i.e. DNS zones). You can set this up once and create many clusters with unqiue names.

```tf
resource "digitalocean_domain" "zone-for-clusters" {
  name       = "do.example.com"
  # Digital Ocean oddly requires an IP here. You may have to delete the A record it makes. :(
  ip_address = "8.8.8.8"
}
```

!!! tip ""
    If you have an existing domain name with a zone file elsewhere, just carve out a subdomain that can be managed on DigitalOcean (e.g. do.mydomain.com) and [update nameservers](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-host-name-with-digitalocean).

#### SSH Fingerprints

DigitalOcean droplets are created with your SSH public key "fingerprint" (i.e. MD5 hash) to allow access. If your SSH public key is at `~/.ssh/id_rsa`, find the fingerprint with,

```bash
ssh-keygen -lf ~/.ssh/id_rsa.pub | awk '{print $2}'
d7:9d:79:ae:56:32:73:79:95:88:e3:a2:ab:5d:45:e7
```

If you use `ssh-agent` (e.g. Yubikey for SSH), find the fingerprint with,

```
ssh-add -l -E md5
2048 MD5:d7:9d:79:ae:56:32:73:79:95:88:e3:a2:ab:5d:45:e7 cardno:000603633110 (RSA)
```

If you uploaded an SSH key to DigitalOcean (not required), find the fingerprint under Settings -> Security. Finally, if you don't have an SSH key, [create one now](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).

### Optional

| Name | Description | Default | Example |
|:-----|:------------|:--------|:--------|
| image | OS image for droplets | "coreos-stable" | coreos-stable, coreos-beta, coreos-alpha |
| controller_count | Number of controllers (i.e. masters) | 1 | 1 |
| controller_type | Digital Ocean droplet size | 2gb | 2gb (min), 4gb, 8gb |
| worker_count | Number of workers | 1 | 3 |
| worker_type | Digital Ocean droplet size | 512mb | 512mb, 1gb, 2gb, 4gb |
| networking | Choice of networking provider | "flannel" | "flannel" |
| pod_cidr | CIDR range to assign to Kubernetes pods | "10.2.0.0/16" | "10.22.0.0/16" |
| service_cidr | CIDR range to assgin to Kubernetes services | "10.3.0.0/16" | "10.3.0.0/24" |

!!! warning
    Do not choose a `controller_type` smaller than `2gb`. The `1gb` droplet is not sufficient for running a controller and bootstrapping will fail.

!!! bug
    Digital Ocean firewalls do not yet support the IP tunneling (IP in IP) protocol used by Calico. You can try using "calico" for `networking`, but it will only work if the cloud firewall is removed (unsafe).
