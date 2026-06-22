# Building Your Own Kubernetes Cluster — DigitalOcean Droplet + Vagrant + k3s + Longhorn

End-to-end: spin up a DigitalOcean droplet, use it as a "VM host" running 3 lightweight Vagrant VMs, install **k3s** (lightweight Kubernetes) across them, add **Longhorn** for persistent storage, then — in the final section — replace the local Vagrant VMs entirely with **real DigitalOcean droplets** as your cluster nodes instead.

---

## Table of Contents

1. Why this stack (k3s vs full Kubernetes, Longhorn vs cloud PVs)
2. Provisioning the DigitalOcean droplet (the Vagrant host)
3. Installing Vagrant + a VM provider on the droplet
4. The Vagrantfile — 3 lightweight VMs
5. Installing k3s — server (control plane) + 2 agents (workers)
6. Verifying the cluster
7. Installing Longhorn (distributed persistent storage)
8. Testing Longhorn with a real PVC
9. Replacing the Vagrant VMs with real DigitalOcean Droplets
10. Tearing down / cost cleanup

---

## 1. Why This Stack

| Choice                                                                         | Why                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **k3s** instead of full `kubeadm` Kubernetes                                   | A single ~70MB binary, bundles `containerd`, a lightweight `etcd` replacement (SQLite by default, or real etcd), and sane defaults — built specifically for resource-constrained environments (this is the same project literally named for "5 less than k8s"). A full `kubeadm` cluster wants more RAM/CPU per node than a single small droplet running 3 nested VMs can comfortably give you.                                                                                                                                                                                                                                                                            |
| **Vagrant**, nested inside one droplet, for the _first_ version of the cluster | Lets you practice the entire "provision N machines → install k3s across them → join them into a cluster" workflow locally/cheaply, with full control over each VM's lifecycle (`vagrant up`/`destroy`/`ssh`), before paying for N separate real droplets.                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Longhorn** for storage                                                       | A distributed block storage system that runs _as Kubernetes workloads themselves_ (DaemonSets/Deployments) — it turns each node's _local_ disk space into replicated, highly-available persistent volumes, without needing a cloud provider's managed disk service. This matters because k3s, by default, only ships with a basic `local-path` storage provisioner — fine for a single-node demo, but it ties a PVC to one specific node's local disk, with **no replication and no portability** if that node goes down. Longhorn is what makes "real" StatefulSet-style workloads (like a Postgres deployment, going back to your earlier notes) survive a node failure. |

**The end state:** a 3-node k3s cluster (1 control plane + 2 workers), each contributing local disk space into a shared, replicated storage pool managed by Longhorn — a genuinely production-shaped architecture, just running on minimal hardware for learning purposes.

---

## 2. Provisioning the DigitalOcean Droplet (the Vagrant Host)

This droplet's only job is to **host the 3 nested VMs** — it needs to be reasonably large itself, since it's running 3 guest VMs inside it.

```bash
doctl compute droplet create vagrant-host \
  --region nyc1 \
  --image ubuntu-22-04-x64 \
  --size s-8vcpu-16gb \
  --ssh-keys <your-ssh-key-fingerprint> \
  --enable-monitoring \
  --wait
```

**Sizing note:** 3 lightweight VMs at ~1 vCPU/1–2GB RAM each need roughly 3–6GB RAM and 3+ vCPUs just for the guests, on top of whatever the host OS and the hypervisor itself need. An `s-8vcpu-16gb` droplet gives comfortable headroom; you can go smaller (`s-4vcpu-8gb`) if you trim each VM down further, but expect things to feel tight.

```bash
# Get the droplet's IP and SSH in
doctl compute droplet list
ssh root@<droplet-ip>
```

Update the system before installing anything:

```bash
apt update && apt upgrade -y
```

---

## 3. Installing Vagrant + a VM Provider on the Droplet

Since this droplet is itself a VM (DigitalOcean runs droplets on KVM), **nested virtualization** is required to run VirtualBox-style VMs inside it — DigitalOcean droplets generally do **not** expose nested virtualization (`/dev/kvm` is typically unavailable), so VirtualBox (which needs real or nested hardware virtualization) usually won't work here.

**The reliable provider in this exact scenario is libvirt with QEMU's software emulation fallback** — slower than hardware-accelerated VMs, but it works without `/dev/kvm`. If your droplet/account _does_ have nested virtualization enabled (some DigitalOcean droplet types support it on request), libvirt with KVM acceleration will be fast and is what we configure below; the same Vagrantfile works either way, just slower without KVM.

```bash
# Check if KVM acceleration is actually available first
ls /dev/kvm 2>/dev/null && echo "KVM available" || echo "No KVM — will run unaccelerated (slower)"
```

```bash
# Install Vagrant
curl -fsSL https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | tee /etc/apt/sources.list.d/hashicorp.list
apt update && apt install -y vagrant

# Install libvirt + QEMU (the VM provider Vagrant will drive)
apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst

# Install the libvirt plugin for Vagrant
vagrant plugin install vagrant-libvirt

# Add your user to the libvirt group (or just continue as root, simplest for a throwaway lab droplet)
usermod -aG libvirt $(whoami)
systemctl enable --now libvirtd
```

---

## 4. The Vagrantfile — 3 Lightweight VMs

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"

  nodes = [
    { name: "k3s-server",  ip: "192.168.56.10", memory: 2048, cpus: 1 },
    { name: "k3s-agent-1", ip: "192.168.56.11", memory: 1536, cpus: 1 },
    { name: "k3s-agent-2", ip: "192.168.56.12", memory: 1536, cpus: 1 },
  ]

  nodes.each do |node|
    config.vm.define node[:name] do |n|
      n.vm.hostname = node[:name]
      n.vm.network "private_network", ip: node[:ip]

      n.vm.provider "libvirt" do |lv|
        lv.memory = node[:memory]
        lv.cpus = node[:cpus]
        lv.nested = true   # harmless if unsupported; helps if your droplet does have KVM access
      end

      n.vm.provision "shell", inline: <<-SHELL
        # Disable swap — k3s/kubelet refuse to run cleanly with swap enabled
        swapoff -a
        sed -i '/swap/d' /etc/fstab

        # Make sure each node resolves the others by hostname
        echo "192.168.56.10 k3s-server"  >> /etc/hosts
        echo "192.168.56.11 k3s-agent-1" >> /etc/hosts
        echo "192.168.56.12 k3s-agent-2" >> /etc/hosts
      SHELL
    end
  end
end
```

```bash
vagrant up
# This brings up all 3 VMs in order; takes a few minutes the first time (box download + provisioning)

vagrant status     # confirm all 3 show "running"
vagrant ssh k3s-server   # SSH into a specific VM by name
```

**Why `192.168.56.x` private networking:** Vagrant's `private_network` gives each VM a stable, predictable IP that the other VMs (and you) can reach directly — this is what lets `k3s-agent-1` reliably find `k3s-server` to join the cluster, regardless of how the host's own networking is configured.

---

## 5. Installing k3s — Server (Control Plane) + 2 Agents (Workers)

### On `k3s-server` (the control plane node)

```bash
vagrant ssh k3s-server
```

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip 192.168.56.10 \
  --advertise-address 192.168.56.10 \
  --tls-san 192.168.56.10
```

This single command installs k3s as a systemd service and starts it immediately as a control-plane node. Once it's running:

```bash
# Get the join token — every agent needs this to authenticate into the cluster
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy that token value — you'll need it for both agent nodes.

```bash
# Confirm the server itself is up
sudo k3s kubectl get nodes
```

### On `k3s-agent-1` and `k3s-agent-2` (worker nodes)

```bash
vagrant ssh k3s-agent-1
```

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.56.10:6443 \
  K3S_TOKEN=<paste-the-token-from-the-server> \
  sh -s - agent --node-ip 192.168.56.11
```

Repeat the same on `k3s-agent-2`, with `--node-ip 192.168.56.12`.

**What's actually happening here, conceptually:** `k3s server` runs the control plane (API server, scheduler, controller-manager, and its embedded datastore) — this is the single source of truth for the cluster's desired state, exactly the role `kube-apiserver` plays in a full Kubernetes install. `k3s agent` runs just the **kubelet** + a lightweight proxy, registering itself with the server and waiting to be assigned Pods — this is precisely the "Node" concept from the earlier k8s notes; you're now building the real infrastructure those notes were abstracting away.

---

## 6. Verifying the Cluster

Back on `k3s-server`:

```bash
sudo k3s kubectl get nodes -o wide
```

Expected output — all 3 nodes, `Ready`:

```
NAME          STATUS   ROLES                  AGE   VERSION
k3s-server    Ready    control-plane,master   5m    v1.30.x+k3s1
k3s-agent-1   Ready    <none>                 2m    v1.30.x+k3s1
k3s-agent-2   Ready    <none>                 2m    v1.30.x+k3s1
```

**Set up `kubectl` access without `sudo k3s kubectl` every time** — copy the kubeconfig and use it with your regular `kubectl` (install `kubectl` separately if it's not already present):

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
# Copy this to your local machine (or wherever you want to manage the cluster from) as ~/.kube/config
# If accessing remotely, replace "127.0.0.1" inside it with the server node's real reachable IP
```

```bash
kubectl get nodes
kubectl get pods -A   # see k3s's own bundled components: coredns, traefik (default ingress), metrics-server
```

k3s ships with **CoreDNS, Traefik (as the default Ingress controller), and metrics-server already installed** — exactly the add-ons your earlier k8s notes had you install manually (e.g. `metrics-server` for HPA support) come bundled out of the box here.

---

## 7. Installing Longhorn (Distributed Persistent Storage)

### Prerequisites on every node (server + both agents)

Longhorn needs a few packages on the underlying OS to manage disks/iSCSI:

```bash
# Run on k3s-server, k3s-agent-1, AND k3s-agent-2
sudo apt update
sudo apt install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid
```

### Install Longhorn itself (run once, from `k3s-server`)

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.0/deploy/longhorn.yaml
```

This deploys Longhorn's own components as Kubernetes workloads — a manager DaemonSet (one Pod per node, exactly like a node-level agent), a UI Deployment, instance managers, and CRDs for `Volume`/`Replica`/`Engine` objects. Watch it come up:

```bash
kubectl get pods -n longhorn-system -w
```

Wait until everything shows `Running`/`1/1` — this can take a few minutes as it pulls several images.

### Make Longhorn the default StorageClass

```bash
kubectl get storageclass
# By default, k3s's own "local-path" class is marked default — Longhorn installs its own "longhorn" class alongside it

# Remove default from local-path, make longhorn the default instead
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass longhorn -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

This is directly relevant to the **PVC/StorageClass concepts from your earlier k8s notes** — `postgres-pvc` omitting `storageClassName` entirely means it'll now dynamically provision via Longhorn instead of k3s's basic local-path provisioner, gaining real replication across nodes instead of being pinned to one node's local disk.

### Access the Longhorn UI (optional, but genuinely useful for visualizing replicas)

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Then browse to `http://localhost:8080` (or `http://<droplet-ip>:8080` if forwarding through your droplet's firewall/SSH tunnel) to see volumes, their replicas, and which nodes they live on.

---

## 8. Testing Longhorn With a Real PVC

```yaml
# longhorn-test-pvc.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

```yaml
# longhorn-test-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test-pod
spec:
  containers:
    - name: writer
      image: busybox
      command:
        [
          "sh",
          "-c",
          "echo 'hello from longhorn' > /data/test.txt && sleep 3600",
        ]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: longhorn-test-pvc
```

```bash
kubectl apply -f longhorn-test-pvc.yml
kubectl apply -f longhorn-test-pod.yml

kubectl get pvc longhorn-test-pvc      # should show STATUS=Bound
kubectl exec longhorn-test-pod -- cat /data/test.txt   # "hello from longhorn"
```

**The real test — kill the node the Pod is on and confirm the data survives:**

```bash
kubectl get pod longhorn-test-pod -o wide   # see which node it's running on, e.g. k3s-agent-1
```

```bash
# From the Vagrant host:
vagrant halt k3s-agent-1    # simulate that node going down
```

```bash
kubectl get pods -o wide -w   # watch Kubernetes reschedule the Pod onto a different node
kubectl exec longhorn-test-pod -- cat /data/test.txt   # data should STILL be there — Longhorn had a replica elsewhere
```

```bash
vagrant up k3s-agent-1   # bring it back when done
```

This is the practical payoff of the whole setup: a Postgres-style stateful workload from your earlier k8s notes can now survive a node failure, something the original `local-path` provisioner (or a plain `hostPath` volume) fundamentally cannot do.

---

## 9. Replacing the Vagrant VMs With Real DigitalOcean Droplets

Everything from Section 5 onward is **identical** whether the 3 machines are nested Vagrant VMs or real droplets — k3s and Longhorn don't care what's underneath them, as long as each node has a real IP, SSH access, and the prerequisite packages. The only things that change are _how you provision the machines_ and _how they reach each other_.

### Step 1 — Create the droplets

```bash
doctl compute droplet create k3s-server \
  --region nyc1 --image ubuntu-22-04-x64 --size s-2vcpu-4gb \
  --ssh-keys <your-ssh-key-fingerprint> --enable-private-networking --wait

doctl compute droplet create k3s-agent-1 \
  --region nyc1 --image ubuntu-22-04-x64 --size s-2vcpu-4gb \
  --ssh-keys <your-ssh-key-fingerprint> --enable-private-networking --wait

doctl compute droplet create k3s-agent-2 \
  --region nyc1 --image ubuntu-22-04-x64 --size s-2vcpu-4gb \
  --ssh-keys <your-ssh-key-fingerprint> --enable-private-networking --wait
```

**`--enable-private-networking` is the direct replacement for Vagrant's `private_network` block** — DigitalOcean gives each droplet in the same region/VPC a private IP they can reach each other through, without traffic going over the public internet (faster, and not exposed externally).

```bash
doctl compute droplet list --format "Name,PublicIPv4,PrivateIPv4"
```

Note each droplet's **private** IP — that's the equivalent of the `192.168.56.x` addresses from the Vagrantfile, used for all inter-node cluster traffic.

### Step 2 — Open the right firewall rules

Unlike the all-local Vagrant setup, real droplets are reachable over the internet by default, so the firewall actually matters now:

```bash
doctl compute firewall create \
  --name k3s-cluster-fw \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0 protocol:tcp,ports:6443,address:0.0.0.0/0 protocol:tcp,ports:1-65535,address:<vpc-cidr> protocol:udp,ports:1-65535,address:<vpc-cidr>" \
  --outbound-rules "protocol:tcp,ports:1-65535,address:0.0.0.0/0 protocol:udp,ports:1-65535,address:0.0.0.0/0" \
  --droplet-ids <server-id>,<agent1-id>,<agent2-id>
```

- **Port `22`** — SSH access for you.
- **Port `6443`** — the k3s/Kubernetes API server, needed for `kubectl` access and for agents to register.
- **All ports, restricted to the VPC's private CIDR** — k3s, Flannel (its default CNI), and Longhorn all need a wide range of node-to-node ports (VXLAN overlay traffic, Longhorn's replica sync, etc.); restricting this range to _only_ your private network (not `0.0.0.0/0`) keeps it from being exposed publicly while still letting the nodes freely talk to each other.

### Step 3 — Install k3s, pointing at private IPs instead of Vagrant's `192.168.56.x`

```bash
ssh root@<k3s-server-public-ip>
```

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip <k3s-server-PRIVATE-ip> \
  --advertise-address <k3s-server-PRIVATE-ip> \
  --tls-san <k3s-server-PUBLIC-ip>
```

**Why `--tls-san` with the public IP this time:** if you ever want to run `kubectl` from your own laptop (not just from inside the cluster's private network), the server's TLS certificate needs to include the public IP as a valid name it can be reached at — this wasn't needed in the Vagrant setup since everything stayed inside the droplet's local network.

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

```bash
ssh root@<k3s-agent-1-public-ip>
curl -sfL https://get.k3s.io | K3S_URL=https://<k3s-server-PRIVATE-ip>:6443 \
  K3S_TOKEN=<token> \
  sh -s - agent --node-ip <k3s-agent-1-PRIVATE-ip>
```

Repeat for `k3s-agent-2` with its own private IP. Everything from Section 6 onward (verifying nodes, installing Longhorn, testing a PVC) works **completely unchanged** — `kubectl get nodes` shows the same 3-node `Ready` cluster, just now backed by real, independently-billed droplets instead of nested VMs sharing one droplet's hardware.

### What you gain and lose compared to the Vagrant version

|                 | Vagrant VMs (1 droplet)                                                 | 3 real droplets                                                                                                                                    |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cost            | One droplet's bill                                                      | Three droplets' bills — more expensive                                                                                                             |
| Fault isolation | All 3 "nodes" die together if the underlying droplet/host fails         | A real outage on one droplet doesn't take down the others — this is what Longhorn's replication is actually protecting against in a meaningful way |
| Performance     | Shared CPU/RAM/disk I/O across nested VMs, plus virtualization overhead | Each node gets its own dedicated resources — noticeably faster, especially disk I/O for Longhorn's replication traffic                             |
| Networking      | Vagrant's private network, fully local                                  | Real DigitalOcean VPC private networking — same conceptual model, genuinely separate machines                                                      |
| Best for        | Learning/practicing the full setup cheaply before committing spend      | An actually meaningful "is this resilient to a node going down" test, and a setup you could realistically keep running                             |

---

## 10. Tearing Down / Cost Cleanup

**Vagrant version:**

```bash
vagrant destroy -f   # removes all 3 VMs and their disks
```

Then optionally destroy the host droplet itself if you're done with it entirely:

```bash
doctl compute droplet delete vagrant-host
```

**Real droplets version:**

```bash
doctl compute droplet delete k3s-server k3s-agent-1 k3s-agent-2
doctl compute firewall delete k3s-cluster-fw
```

DigitalOcean bills droplets by the hour while they exist (running or stopped) — deleting (not just powering off) is what actually stops billing.
