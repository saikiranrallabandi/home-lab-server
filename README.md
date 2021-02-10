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


