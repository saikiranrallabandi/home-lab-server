# home-lab-server

#### Creating Guest VMs

```
$ kvm-install-vm help create
NAME
    kvm-install-vm create [COMMANDS] [OPTIONS] VMNAME

DESCRIPTION
    Create a new guest domain.

COMMANDS
    help - show this help

OPTIONS
    -a          Autostart             (default: false)
    -b          Bridge                (default: virbr0)
    -c          Number of vCPUs       (default: 1)
    -d          Disk Size (GB)        (default: 10)
    -D          DNS Domain            (default: example.local)
    -f          CPU Model / Feature   (default: host)
    -g          Graphics type         (default: spice)
    -h          Display help
    -i          Custom QCOW2 Image
    -k          SSH Public Key        (default: $HOME/.ssh/id_rsa.pub)
    -l          Location of Images    (default: $HOME/virt/images)
    -L          Location of VMs       (default: $HOME/virt/vms)
    -m          Memory Size (MB)      (default: 1024)
    -M          Mac address           (default: auto-assigned)
    -p          Console port          (default: auto)
    -s          Custom shell script
    -t          Linux Distribution    (default: centos8)
    -T          Timezone              (default: US/Eastern)
    -u          Custom user           (default: $USER)
    -y          Assume yes to prompts (default: false)
    -n          Assume no to prompts  (default: false)
    -v          Be verbose

DISTRIBUTIONS
    NAME            DESCRIPTION                         LOGIN
    amazon2         Amazon Linux 2                      ec2-user
    centos8         CentOS 8                            centos
    centos7         CentOS 7                            centos
    centos7-atomic  CentOS 7 Atomic Host                centos
    centos6         CentOS 6                            centos
    debian9         Debian 9 (Stretch)                  debian
    debian10        Debian 10 (Buster)                  debian
    fedora29        Fedora 29                           fedora
    fedora29-atomic Fedora 29 Atomic Host               fedora
    fedora30        Fedora 30                           fedora
    fedora31        Fedora 31                           fedora
    fedora32        Fedora 32                           fedora
    opensuse15      OpenSUSE Leap 15.2                  opensuse
    ubuntu1604      Ubuntu 16.04 LTS (Xenial Xerus)     ubuntu
    ubuntu1804      Ubuntu 18.04 LTS (Bionic Beaver)    ubuntu
    ubuntu2004      Ubuntu 20.04 LTS (Focal Fossa)      ubuntu

EXAMPLES
    kvm-install-vm create foo
        Create VM with the default parameters: CentOS 8, 1 vCPU, 1GB RAM, 10GB
        disk capacity.

    kvm-install-vm create -c 2 -m 2048 -d 20 foo
        Create VM with custom parameters: 2 vCPUs, 2GB RAM, and 20GB disk
        capacity.

    kvm-install-vm create -t debian9 foo
        Create a Debian 9 VM with the default parameters.

    kvm-install-vm create -T UTC foo
        Create a default VM with UTC timezone.
```

#### Deleting a Guest Domain

```
$ kvm-install-vm help remove
NAME
    kvm-install-vm remove [COMMANDS] VMNAME

DESCRIPTION
    Destroys (stops) and undefines a guest domain.  This also remove the
    associated storage pool.

COMMANDS
    help - show this help

EXAMPLE
    kvm-install-vm remove foo
        Remove (destroy and undefine) a guest domain.  WARNING: This will
        delete the guest domain and any changes made inside it!
```

#### Attaching a new disk

```
$ kvm-install-vm help attach-disk
NAME
    kvm-install-vm attach-disk [OPTIONS] [COMMANDS] VMNAME

DESCRIPTION
    Attaches a new disk to a guest domain.

COMMANDS
    help - show this help

OPTIONS
    -d SIZE     Disk size (GB)
    -f FORMAT   Disk image format       (default: qcow2)
    -s IMAGE    Source of disk device
    -t TARGET   Disk device target

EXAMPLE
    kvm-install-vm attach-disk -d 10 -s example-5g.qcow2 -t vdb foo
        Attach a 10GB disk device named example-5g.qcow2 to the foo guest
        domain.
```

### Setting Custom Defaults

Copy the `.kivrc` file to your $HOME directory to set custom defaults.  This is
convenient if you find yourself repeatedly setting the same options on the
command line, like the distribution or the number of vCPUs.

Options are evaluated in the following order:

- Default options set in the script
- Custom options set in `.kivrc`
- Option flags set on the command line

### K8S PRE SETUP

## Role info

> This playbook is not for fully setting up a Kubernetes Cluster.

It only helps you automate the standard Kubernetes bootstrapping pre-reqs.

## Supported OS

- CentOS 7

## Tasks in the role

This role contains tasks to:

- Install basic packages required
- Setup standard system requirements - Disable Swap, Modify sysctl, Disable SELinux
- Install and configure a container runtime of your Choice - cri-o, Docker, Containerd
- Install the Kubernetes packages - kubelet, kubeadm and kubectl
- Configure Firewalld on Kubernetes Master and Worker nodes

## How to use this role

- Clone the Project:

```
$ git clone https://github.com/jmutai/k8s-pre-bootstrap.git
```

- Update your inventory, e.g:

```
$ vim hosts
[k8s-nodes]
172.21.200.10
172.21.200.11
172.21.200.12
```

- Update variables in playbook file

```
$ vim k8s-prep.yml
---
- name: Setup Proxy
  hosts: k8s-nodes
  remote_user: root
  become: yes
  become_method: sudo
  #gather_facts: no
  vars:
    k8s_cni: calico                                      # calico, flannel
    container_runtime: docker                            # docker, cri-o, containerd
    configure_firewalld: true                            # true / false
    # Docker registry
    setup_proxy: false                                   # Set to true to configure proxy
    proxy_server: "proxy.example.com:8080"               # Proxy server address and port
    docker_proxy_exclude: "localhost,127.0.0.1"          # Adresses to exclude from proxy
  roles:
    - kubernetes-bootstrap
```

If you are using non root remote user, then set username and enable sudo:

```
become: yes
become_method: sudo
```

To enable proxy, set the value of `setup_proxy` to `true` and provide proxy details.

## Running Playbook

Once all values are updated, you can then run the playbook against your nodes.

**NOTE**: For firewall configuration to open relevant ports for master and worker nodes, a pattern in hostname is required.

Check file:

```
$ vim roles/kubernetes-bootstrap/tasks/configure_firewalld.yml
....
- name: Configure firewalld on master nodes
  firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    state: enabled
  with_items: '{{ k8s_master_ports }}'
  when: "'master' in ansible_hostname"

- name: Configure firewalld on worker nodes
  firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    state: enabled
  with_items: '{{ k8s_worker_ports }}'
  when: "'node' in ansible_hostname"
```

If your master nodes doesn't contain `master` and nodes doesn't have `node` as part of hostname, update the file to reflect your naming pattern. My nodes are named like below:

```
k8smaster01
k8snode01
k8snode02
k8snode03
```

Playbook executed as root user - with ssh key:

```
$ ansible-playbook -i hosts k8s-prep.yml
```

Playbook executed as root user - with password:

```
$ ansible-playbook -i hosts k8s-prep.yml --ask-pass
```

Playbook executed as sudo user - with password:

```
$ ansible-playbook -i hosts k8s-prep.yml --ask-pass --ask-become-pass
```

Playbook executed as sudo user - with ssh key and sudo password:

```
$ ansible-playbook -i hosts k8s-prep.yml --ask-become-pass
```

Playbook executed as sudo user - with ssh key and passwordless sudo:

```
$ ansible-playbook -i hosts k8s-prep.yml --ask-become-pass
```

Execution should be successful without errors:

```
TASK [kubernetes-bootstrap : Reload firewalld] *********************************************************************************************************
changed: [k8smaster01]
changed: [k8snode01]
changed: [k8snode02]

PLAY RECAP *********************************************************************************************************************************************
k8smaster01                : ok=23   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
k8snode01                  : ok=23   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
k8snode02                  : ok=23   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
```


##### HPA K8S CLUSTER

jupyter
ssh homeserver@192.168.0.24 -L 8888:127.0.0.1:8888
jupyter notebook --ip 192.168.0.24 --port 8888


---

IPTABLES ACL 
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
iptables -t nat -A PREROUTING -p tcp --dport 9867 -j DNAT --to 192.168.122.200:22
```
sudo iptables -I FORWARD -o virbr0 -d 192.168.122.200 -j ACCEPT
sudo iptables -t nat -I PREROUTING -p tcp --dport 9000 -j DNAT --to 192.168.122.200:22
sudo iptables -I FORWARD -o virbr0  -d  192.168.122.200 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 192.168.122.0/24 -j MASQUERADE
sudo iptables -A FORWARD -o virbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i virbr0 -o em1 -j ACCEPT
sudo iptables -A FORWARD -i virbr0 -o lo -j ACCEPT
sudo systemctl enable iptables
sudo service iptables save

---
How to Clear RAM Memory Cache, Buffer and Swap Space on Linux

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


https://www.tecmint.com/clear-ram-memory-cache-buffer-and-swap-space-on-linux/




---
How to Setup Kubernetes(k8s) Cluster in HA with Kubeadm

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Minimum requirements for setting up Highly K8s cluster

- Install Kubeadm, kubelet and kubectl on all master and worker Nodes
- Network Connectivity among master and worker nodes
- Internet Connectivity on all the nodes
- Root credentials or sudo privileges user on all nodes
Let’s jump into the installation and configuration steps

Step 1) Set Hostname and add entries in /etc/hosts file



Step 2) Install and Configure Keepalive and HAProxy on all master / control plane nodes


```
$ sudo yum install haproxy keepalived -y
```

Configure Keepalived on k8s-master-1 first, create check_apiserver.sh script will the following content,

```
[kadmin@k8s-master-1 ~]$ sudo vi /etc/keepalived/check_apiserver.sh
#!/bin/sh
APISERVER_VIP=192.168.1.45
APISERVER_DEST_PORT=6443

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
```

save and exit the file.

Set the executable permissions



```
sudo chmod +x /etc/keepalived/check_apiserver.sh
```

Take the backup of keepalived.conf file and then truncate the file.

```
[kadmin@k8s-master-1 ~]$ sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf-org
[kadmin@k8s-master-1 ~]$ sudo sh -c '> /etc/keepalived/keepalived.conf'
```

Now paste the following contents to /etc/keepalived/keepalived.conf file

```
[kadmin@k8s-master-1 ~]$ sudo vi /etc/keepalived/keepalived.conf
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface enp0s3
    virtual_router_id 151
    priority 255
    authentication {
        auth_type PASS
        auth_pass P@##D321!
    }
    virtual_ipaddress {
        192.168.1.45/24
    }
    track_script {
        check_apiserver
    }
}
```

Note: Only two parameters of this file need to be changed for master-2 & 3 nodes. State will become SLAVE for master 2 and 3, priority will be 254 and 253 respectively.

Configure HAProxy on k8s-master-1 node, edit its configuration file and add the following contents:



```
[kadmin@k8s-master-1 ~]$ sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-org
```

Remove all lines after default section and add following lines


```
[kadmin@k8s-master-1 ~]$ sudo vi /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# apiserver frontend which proxys to the masters
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443
    mode tcp
    option tcplog
    default_backend apiserver
#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server k8s-master-1 192.168.1.40:6443 check
        server k8s-master-2 192.168.1.41:6443 check
        server k8s-master-3 192.168.1.42:6443 check
```

Save and exit the file


Now copy theses three files (check_apiserver.sh , keepalived.conf and haproxy.cfg) from k8s-master-1 to k8s-master-2 & 3

Run the following for loop to scp these files to master 2 and 3


```
[kadmin@k8s-master-1 ~]$ for f in k8s-master-2 k8s-master-3; do scp /etc/keepalived/check_apiserver.sh /etc/keepalived/keepalived.conf root@$f:/etc/keepalived; scp /etc/haproxy/haproxy.cfg root@$f:/etc/haproxy; done
```

Note: Don’t forget to change two parameters in keepalived.conf file that we discuss above for k8s-master-2 & 3

In case firewall is running on master nodes then add the following firewall rules on all three master nodes


```
sudo yum install firewalld -y
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
sudo firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload
```

Now Finally start and enable keepalived and haproxy service on all three master nodes using the following commands :

```
sudo systemctl enable keepalived --now
sudo systemctl enable haproxy --now
```

Once these services are started successfully, verify whether VIP (virtual IP) is enabled on k8s-master-1 node because we have marked k8s-master-1 as MASTER node in keepalived configuration file.



Perfect, above output confirms that VIP has been enabled on k8s-master-1.

Firewall Rules for Master Nodes:

In case firewall is running on master nodes, then allow the following ports in the firewall,



```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
sudo modprobe br_netfilter
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"
```

Firewall Rules for Worker nodes:

In case firewall is running on worker nodes, then allow the following ports in the firewall on all the worker nodes



Run the following commands on all the worker nodes,

```
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
sudo modprobe br_netfilter
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"

```

Pre-requiste install packages

```
ansible-playbook -i hosts k8s-prep.yml
```

Step 6) Initialize the Kubernetes Cluster from first master node


```
sudo kubeadm init --control-plane-endpoint "vip-k8s-master:8443" --upload-certs
```

In above command, –control-plane-endpoint set dns name and port for load balancer (kube-apiserver), in my case dns name is “vip-k8s-master” and port is “8443”, apart from this ‘–upload-certs’ option will share the certificates among master nodes automatically,

Output of k

ubeadm command would be something like below:

```
[kadmin@k8s-master-2 ~]$ sudo kubeadm join vip-k8s-master:8443 --token tun848.2hlz8uo37jgy5zqt  --discovery-token-ca-cert-hash sha256:d035f143d4bea38d54a3d827729954ab4b1d9620631ee330b8f3fbc70324abc5 --control-plane --certificate-key a0b31bb346e8d819558f8204d940782e497892ec9d3d74f08d1c0376dc3d3ef4
```

```
kubeadm join k8smaster1.saikiranrallanandi.com:6443 --token utmpm4.xhd6vbd9wx3nrl7l \
    --discovery-token-ca-cert-hash sha256:a98f59f62e25786bac0cba11b2a1c61777bab72669313853e64eadbae6466a02 \
    --control-plane --certificate-key 0376850a02dc671549dcaba41623fbe8ab90db5d40dfd80b2acb4fa840324086
```