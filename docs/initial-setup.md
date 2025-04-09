# Initial Setup (Getting Started)
I have 4 nodes in my HomeLab
| Device                      | Name      | Disk Size |  Ram  | CPU        | Operating System  | Purpose              |
|-----------------------------|-----------|-----------|-------|------------|-------------------|----------------------|
| Custom Made PC              | rel       | 1TB NVMe  | 64GB  | 24C/32T    |Ubuntu 22.04.5 LTS | Kubernetes Worker    |
| Mini PC 1                   | mini1     | 64GB SSD  | 16GB  | 4C/4T      |Ubuntu 22.04.5 LTS | Kubernetes Master 1  |
| Mini PC 2                   | mini2     | 64GB SSD  | 16GB  | 4C/4T      |Ubuntu 22.04.5 LTS | Kubernetes Master 2  |
| Mini PC 3                   | mini3     | 64GB SSD  | 16GB  | 4C/4T      |Ubuntu 22.04.5 LTS | Kubernetes Master 3  |

---

<br>

## Setting up the Cluster
I am using Kubespray to deploy Kubernetes in my HomeLab. My custom variables for the cluster & hosts (inventory) file can be found under `kubernetes/kubespray`

### Pre Settings for all nodes
Before running kubespray ansible, all of the nodes should be prepared with the following steps: 
- Create a root password (same for all nodes in my case)
- Update `/etc/ssh/sshd_config` file and uncomment the `PermitRootLogin yes` option within the file (for root SSH into all nodes)
-  Restart sshd service using systemctl

### Running Kubespray 
Running the docker command for kubespray
```bash
$ docker run --rm -it --mount type=bind,source="$(pwd)"/hosts.yml,dst=/inventory --mount type=bind,source="$(pwd)"/vars.yml,dst=/vars quay.io/kubespray/kubespray:v2.27.0 bash
```
<br>

Once inside the Kubespray container, run the following ansible command to verify if the container can reach to all nodes via SSH
```bash
$ ansible all -i /inventory -m ping -k
SSH password: 
[WARNING]: Skipping callback plugin 'ara_default', unable to load
mini1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
mini2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
mini3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
rel | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
<br>

Finally, run the kubespray `cluster.yml` file to install kubernetes
```bash
$ ansible-playbook -e @/vars -i /inventory cluster.yml -k
```

<br>
