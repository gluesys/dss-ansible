# TESS Deploy README.md

## Overview

Ansible automation for configuration, deployment and orchestration of TESS software.

### Ansible Host Setup

On a linux host, on the same management network as all hosts in the TESS cluster, install the latest version of Ansible version 2.9 using python3.
Note that Ansible 2.10 or later is not yet supported.
**Do not use a package-manager version of Ansible.**

    yum install -y python3
    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    python3 get-pip.py
    python3 -m pip install "ansible>=2.9,<2.10"

Additionally, ensure that the following python modules are installed on the Ansible host:

    python3 -m pip install "netaddr>=0.8.0"
    python3 -m pip install "jinja2>=2.8"
    python3 -m pip install "paramiko>=2.7.1"
    python3 -m pip install "jmespath>=0.10.0"

The above dependencies will be validated on runtime, and if not met, will fail assertion, preventing Ansible playbook execution.

### TESS Host Setup

#### Add Ansible User

A user should be provisioned on all hosts in the cluster with "NOPASSWD" sudo access.
Ansible will use this account for all automated configuration.
Alternatively, the root user can be used instead, but should be discouraged.

#### Enable SSH access

Ensure the Ansible host can access all hosts in the cluster via SSH.
The SSH service should be started and enabled on all hosts.

#### Generate SSH key

Generate an SSH key on Ansible host, unless already generated.

Example command:

    [user@host ~] ssh-keygen
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/ylo/.ssh/id_rsa):
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in id_rsa.
    Your public key has been saved in id_rsa.pub.
    The key fingerprint is:
    SHA256:GKW7yzA1J1qkr1Cr9MhUwAbHbF2NrIPEgZXeOUOz3Us user@host
    The key's randomart image is:
    +---[RSA 2048]----+
    |.*++ o.o.        |
    |.+B + oo.        |
    | +++ *+.         |
    | .o.Oo.+E        |
    |    ++B.S.       |
    |   o * =.        |
    |  + = o          |
    | + = = .         |
    |  + o o          |
    +----[SHA256]-----+

#### Copy Public Key for Passwordless SSH Authentication

Copy the SSH public key to each of the hosts using the shh-copy-id command:

    ssh-copy-id ansible@testhost01.domain.com

To copy keys to 10 hosts automatically, use `sshpass` in a bash loop, for example:

    for i in {01..10}; do sshpass -p "ansible" ssh-copy-id -o StrictHostKeyChecking=no ansible@testhost${i}.domain.com; done

#### Validate Passwordless SSH Authentication

To validate that the public key was copied successfully, verify that you can SSH to each host without providing a password, from the Ansible host.

### Prepare Inventory File

Prior to creating your inventory file for your cluster, it is necessary to have some understanding of Ansible inventory file creation, as well as Ansible variables.

Please reference the official documentation:

<https://docs.ansible.com/ansible/2.9/user_guide/intro_inventory.html>
<https://docs.ansible.com/ansible/2.9/user_guide/playbooks_variables.html>

#### Example Inventory File

Configure an Ansible inventory file for the cluster.
Below is an example inventory file defining a cluster of 10 TESS VM hosts in the [servers] group, with one also in the [clients] group.

    [servers]
    msl-ssg-vm01.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::1']" rocev2_ip_list="['192.168.199.1']"
    msl-ssg-vm02.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::2']" rocev2_ip_list="['192.168.199.2']"
    msl-ssg-vm03.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::3']" rocev2_ip_list="['192.168.199.3']"
    msl-ssg-vm04.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::4']" rocev2_ip_list="['192.168.199.4']"
    msl-ssg-vm05.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::5']" rocev2_ip_list="['192.168.199.5']"
    msl-ssg-vm06.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::6']" rocev2_ip_list="['192.168.199.6']"
    msl-ssg-vm07.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::7']" rocev2_ip_list="['192.168.199.7']"
    msl-ssg-vm08.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::8']" rocev2_ip_list="['192.168.199.8']"
    msl-ssg-vm09.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::9']" rocev2_ip_list="['192.168.199.9']"
    msl-ssg-vm10.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::10']" rocev2_ip_list="['192.168.199.10']"

    [clients]
    msl-ssg-vm11.msl.lab

    [all:vars]
    ansible_user=ansible
    target_fw_version=1.0
    dss_target_mode=kv_block_vm

#### Inventory Groups

Your inventory file should contain hosts in the following groups:

* [servers]
  * Hosts running the TESS collocated software stack:
    * Target
    * NVMe Driver
    * MinIO
* [clients]
  * Hosts running client software (can also be members of [servers] group)
    * AI Benchmark
    * S3 Benchmark
    * Datamover / Client Library
* [ufm_hosts]
  * Optional group of hosts used to deploy Universal Fabric Manager, used for stats collection to Graphite server
    * Can be a member of [servers] or [clients] groups, or can be a lightweight, standalone CentOS host or VM
* [onyx]
  * Optional group of Mellanox Onyx switches your hosts are connected to.
    * For Samsung internal use only. - Not supported!

#### Host Networking vars

##### Ansible User

All hosts should have an `ansible_user` var defined, unless the user running the Ansible Playbooks on the Ansible Host is also the same username on your TESS hosts.
If all hosts in your inventory use the same username, you can achieve this by specifying a group var in your inventory:

    [all:vars]
    ansible_user=ansible

The `ansible_user` var can also be defined as a host var in your inventory:

    [servers]
    msl-ssg-vm01.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::1']" rocev2_ip_list="['192.168.199.1']" ansible_user=foo
    msl-ssg-vm02.msl.lab tcp_ip_list="['fd81:dead:beef:cafe:ff::2']" rocev2_ip_list="['192.168.199.2']" ansible_user=bar

##### Front End Traffic: `tcp_ip_list` var

All hosts in the [servers] and [clients] groups must have `tcp_ip_list` and `rocev2_ip_list` lists defined.

`tcp_ip_list` represents a list of IPV4 addresses, IPV6 addresses, or resolvable hostnames to be used for front-end traffic in the TESS cluster.
This includes all MinIO client, AI Benchmark, S3 Benchmark, and Datamover traffic.
All hosts in [servers] and [clients] groups must have a populated `tcp_ip_list` list var defined.  Please reference the above sample inventory for example.
While not optimal, it is possible to use the same IPs defined in the `rocev2_ip_list` for `tcp_ip_list`.
If IP addresses rather than hostnames are provided, Ansible will assign a TCP Alias for each provided IP in `tcp_ip_list`.
These TCP Aliases will be set in `/etc/hosts` of each host in the [servers] and [clients] groups.
The TCP alias of each IP will be used for all front-end communcation.
If you have already created your own TCP aliases (in `/etc/hosts` or through DNS), provide them in `tcp_ip_list` instead of actual IPs.

##### Back End Traffic: `rocev2_ip_list` var

`rocev2_ip_list` represents a list of IPV4 addresses only, to be used for back-end traffic on the TESS cluster.
These IPs must support RoCEv2 protocol. The TESS target will fail to start unless it can bind to RoCEv2 for each IP.
All hosts in the [servers] group must have a populated `rocev2_ip_list` list var defined. Please reference the above sample inventory for example.
All hosts in the [clients] group should have an empty list set for `rocev2_ip_list`, unless they also appear in the [servers] group.
For stand-alone clients that do not appear in the [servers] group, you can achieve this by specifying a group var in your inventory:

    [clients:vars]
    rocev2_ip_list=[]

Note that if a host appears in multiple groups, host vars only need to be defined a single time. It is not necessary to define the same vars each time the host appears in your inventory.

#### Target-specific Variables

##### Number of Subsystems

By default, each host in the [servers] group will export a single NVMeOF subsystem per physical CPU socket, assuming there is at least one RoCEv2 IP NUMA-assigned to each socket.
It is critical that there be an adequate number of subsystems to satisfy MinIO's erasure coding criteria.
For details, please see <https://docs.min.io/minio/baremetal/concepts/erasure-coding.html>

There must be a minimum of 4 subsystems in your cluster in order for MinIO to start. If this condition is not met, Ansible will fail an assertion and alert the user.

Since one MinIO instance is started per TCP IP on each host, each instance must have 4 subsystems. Assuming TCP IPs are on separate NUMA nodes,
there must be a minimum of 4 subsystems spread across NUMA nodes on all hosts in the cluster.

Example of minimally-viable multi-NUMA node configuration:

* 4 hosts in [servers] group
* 2 NUMA nodes each host
* 2 RoCEv2 IPs
* 2 TCP IPs

In the above cluster, with default settings:

* 2 subsystems will be created on each host (one per NUMA node), for a total 8 subsystems
* 2 MinIO instances will be spawned on each host (one per TCP IP)
* Each MinIO instance will connect to 4 NVMeOF subsystems across all hosts in the cluster, which belong to the same NUMA node as each MinIO TCP IP

If you wish to deploy a cluster with sub-optimal configuration (not enough hosts to satisfy MinIO EC), this can be achieved by specifying the `num_subsystems` var for each host.

Example of sub-optimal, single-NUMA node configuration:

* 2 hosts in [servers] group
* 1 NUMA node per host
* 1 RoCEv2 IP
* 1 TCP IP

In the above cluster, with default settings:

* 1 subsystem will be created on each host, for a total of 2 subsystems
* 1 MinIO instance will be spanwed on each host
* Each MinIO instance will connect to 2 NVMeOF subsystems across both hosts, but will fail to meet the EC criteria

To satisfy the EC criteria, each host can export an additional subsystem, provided there are at least 2 NVMe SSDs per host (minimum of one per subsystem).
To achieve this, you can specify the `num_subsystems` for each host, or as a group var in your inventory:

    [servers:vars]
    num_subsystems=2

##### Target Mode

The `dss_target_mode` var must be specified for all hosts in the [servers] group.

This variable can be set to one of four values:

* kv
  * SSD with KV firmware, exports a KV NVMeOF subsystem
* kv_block_vm
  * SSD with block firmware, using BlobFS for KV datastore. Reduced requirements for VMs or sub-optimal physical hosts (reduced cores, less than 512GB memory)
* kv_block_perf
  * SSD with block firmware, using BlobFS for KV datastore. Optimal requirements for high performance hosts.
* block
  * SSD with block firmware, exports a block NVMeOF subsystem (not supported)

This var can be specified as a group var in your inventory file:

    [servers:vars]
    dss_target_mode=kv_block_perf

#### Target Firmware Version

All hosts in the [servers] group must have a `target_fw_version` var defined.
This var should contain a string representing the firmware version of the block NVMe SSDs you wish to use for the TESS datastore on each host.
If more than one model of SSD is used, you may provide a space-separated list.
This var may be defined as a group var in your inventory:

    [servers:vars]
    target_fw_version=EDA53W0Q EPK9AB5Q

Note that virtual disks on VMware hosts using the Virtual NVMe controller have a firmare version `1.0`. In this scenario, specify in your inventory file:

    [servers:vars]
    target_fw_version=1.0

#### Multi-Cluster Inventory Configuration

Due to MinIO's limitation of supporting a maximum of 36 disks (NVMeOF subsystems), it may be necessary to deploy your hosts in a multi-cluster configuration for large-scale deployments.
It is possible to deploy multiple sets of logically isolated MinIO clusters, but still be able to distribute PUT data across clusters, and access GET data across clusters, using
the DSS Client Library and supported applications.

To specify a multi-cluster environment, specify a `cluster_num` var for each host in your inventory.
Hosts with a matching `cluster_num` string value will be grouped together as a single cluster.

Example [servers] group with 3 logical clusters defined:

    [servers]
    msl-ssg-vm01.msl.lab tcp_ip_list="['msl-ssg-vm1-v6.msl.lab']" rocev2_ip_list="['192.168.200.1']" cluster_num=0
    msl-ssg-vm02.msl.lab tcp_ip_list="['msl-ssg-vm2-v6.msl.lab']" rocev2_ip_list="['192.168.200.2']" cluster_num=0
    msl-ssg-vm03.msl.lab tcp_ip_list="['msl-ssg-vm3-v6.msl.lab']" rocev2_ip_list="['192.168.200.3']" cluster_num=0
    msl-ssg-vm04.msl.lab tcp_ip_list="['msl-ssg-vm4-v6.msl.lab']" rocev2_ip_list="['192.168.200.4']" cluster_num=0
    msl-ssg-vm05.msl.lab tcp_ip_list="['msl-ssg-vm5-v6.msl.lab']" rocev2_ip_list="['192.168.200.5']" cluster_num=two
    msl-ssg-vm06.msl.lab tcp_ip_list="['msl-ssg-vm6-v6.msl.lab']" rocev2_ip_list="['192.168.200.6']" cluster_num=two
    msl-ssg-vm07.msl.lab tcp_ip_list="['msl-ssg-vm7-v6.msl.lab']" rocev2_ip_list="['192.168.200.7']" cluster_num=two
    msl-ssg-vm08.msl.lab tcp_ip_list="['msl-ssg-vm8-v6.msl.lab']" rocev2_ip_list="['192.168.200.8']" cluster_num=two
    msl-ssg-vm09.msl.lab tcp_ip_list="['msl-ssg-vm9-v6.msl.lab']" rocev2_ip_list="['192.168.200.9']" cluster_num="third cluster"
    msl-ssg-vm10.msl.lab tcp_ip_list="['msl-ssg-vm10-v6.msl.lab']" rocev2_ip_list="['192.168.200.10']" cluster_num="third cluster"
    msl-ssg-vm11.msl.lab tcp_ip_list="['msl-ssg-vm11-v6.msl.lab']" rocev2_ip_list="['192.168.200.11']" cluster_num="third cluster"
    msl-ssg-vm12.msl.lab tcp_ip_list="['msl-ssg-vm12-v6.msl.lab']" rocev2_ip_list="['192.168.200.12']" cluster_num="third cluster"

Note that MinIO EC limitations apply for each logical cluster in your inventory.

## Configure Hosts and Deploy TESS Software Stack (Quick-Start)

### Initial Host Configuration

Configure VMs (or hosts when you wish to not use OFED):
  
    ansible-playbook -i your_inventory playbooks/configure_vms.yml

Configure Physical Hosts (using OFED):
  
    ansible-playbook -i your_inventory playbooks/configure_hosts.yml

### Pre-Deployment Network Test

Validate Network Settings (cross-ping all TCP and RoCEv2 endpoints, perform ib_read_bw test):
  
    ansible-playbook -i your_inventory playbooks/test_network.yml

### Deploy TESS Software Stack

Deploy TESS: Deploy and start TESS Software Stack:
  
    ansible-playbook -i your_inventory playbooks/deploy_dss_software.yml

### Upload NFS Data to MinIO Using Datamover

Datamover PUT dry-run:
  
    ansible-playbook -i your_inventory playbooks/start_datamover.yml -e "datamover_dryrun=true"

Datamover PUT:
  
    ansible-playbook -i your_inventory playbooks/start_datamover.yml

### Complete Playbook Documentation

#### playbooks/cleanup_dss_ai_benchmark.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook in the event that "start_dss_ai_benchmark" has failed.
This playbook will terminate stuck warp processes.
Additionally, kernel packet pacing will be removed from each server.

#### playbooks/cleanup_dss_minio.yml

Execute this playbook to cleanup MinIO object store metadata.
This playbook will stop MinIO, and execute cleanup from a single node of each cluster in your inventory.
This will effectively remove any metadata at the root level of the object store.

Note that this will create the appearance of a reformated object store, but space will not be reclaimed.
Note that if a bucket is re-created after executing cleanup, its child objects will become accessible again.

#### playbooks/configure_hosts.yml

Execute this playbook to configure hosts prior to deploying DSS / TESS software.
This playbook will deploy custom kernel, install YUM / python dependencies, and configure OFED.
Note that presently, only versions of CentOS between 7.4 and 7.8 are supported

#### playbooks/configure_inbox_infiniband.yml

Execute this playbook to deploy the in-box Infiniband Support group.
This playbook is intended to be used to configure VMs or hosts where OFED is not desired.
Note that if configuring a host with inbox Infiniband Support, OFED must be removed from the system first.
Note that hosts configured with inbox Infiniband Support must be configured in your inventory with
"tcp_ip_list" and "rocev2_ip_list" lists populated. See README.md for additional details on inventory configuration.

#### playbooks/configure_vlans.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to automatically configure VLANs in Mellanox / Onyx environment.

Hosts that are configured with this playbook do not need to have "tcp_ip_list" and "rocev2_ip_list" lists defined in inventory.
If these vars are not defined, Ansible will attempt to automatically discover IP / VLAN configuration, as configured by this playbook.

Environment MUST have Mellanox Onyx switch configured in inventory.
Onyx Switch(es) must be included under the [onyx] group in your inventory file.
Switch credentials must be provided using "ansible_user" and "ansible_ssh_pass" vars for each Onyx host or the entire [onyx] group.
Key-based authentication is not possible with Onyx.

It is critical to review the "rocev2_vlans" and "tcp_vlans" vars under "group_vars/all.yml" to auto-configure VLANs on Onyx.
To make changes to the default values, uncomment these vars and make changes to "group_vars/all.yml" or add them to your inventory file.
There must be an even number of physical interface ports available, with each pair of ports corresponding to one TCP and one RoCEv2 VLAN/IP.
IPV4 last octet / IPV6 last hextet are automatically derived based on the presence of a digit appended to each hostname in your inventory.
If a unique number cannot be automatically derived from each hostname in your inventory, a number will automatically be assigned to each host
in your inventory, starting with "1".
You can specify a static last octet to each host by assigning the variable, "last_octet" for each host in your inventory file.
Additionally, you can offset the automatically derived last octet for a host, or a group, by assigning a "last_octet_offset" var for each host or group.
For example, given two hosts in your inventory: "server_01" and "client_01"
You can assign "last_octet_offset=100" to "client_01", which will result in "client_01" having a derived last octet of "101"

VLAN IDs for all Onyx switches, clients, and servers must be assigned in your inventory file using "tcp_vlan_id_list" and "rocev2_vlan_id_list" vars.
The VLAN IDs provided in these list must correspond to the VLAN IDs provided in "rocev2_vlans" and "tcp_vlans" lists, as mentioned above.
If using multiple Onyx switches, it is thus possible to spread your VLANs across multiple switches.
Note that if a host and switch both are tagged with a VLAN ID, it is expected that there is a physical link between this switch and host.
If no link is detected, the playbook fail on this assertion.

Note that it is possible to assign multiple VLANs to a single physical link using the "num_vlans_per_port" var as a host / group var.
For example, a host with "num_vlans_per_port=2" and 2 physical ports will allow the use of 4 VLANs (2 TCP and 2 RoCEv2)

#### playbooks/configure_vms.yml

Execute this playbook to configure VMs, or other hosts using the in-box Infiniband Support group.
This playbook is intended to be used to configure VMs or hosts where OFED is not desired.

Note that if configuring a host with inbox Infiniband Support, OFED must be removed from the system first.
Note that hosts configured with inbox Infiniband Support must be configured in your inventory with
"tcp_ip_list" and "rocev2_ip_list" lists populated. See README.md for additional details on inventory configuration.

#### playbooks/debug_dss_software.yml

Execute this playbook to run a series of basic status debug tests .
This playbook will perform the following actions:

* Get a count of running MinIO instances on each host
* Get a count of running target instances on each host
* Search for errors in all MinIO logs across all hosts
* Search for errors in all target logs across all hosts

#### playbooks/deploy_datamover.yml

Execute this playbook to deploy the datamover, client library, and their dependencies.
Artifacts are deployed to hosts under the [clients] group.
Note that it is possible for hosts to appear under both the [clients] and [servers] groups.
Hosts under the [clients] group will be used for datamover distributed operations.
This playbook will also create configuration files for client library and datamover, based on hosts that appear in your inventory.
Please review "Datamover Settings" under "group_vars/all.yml" if you wish to adjust the default settings of the datamover.
Uncomment vars with new values, or add them to your inventory file.
It is critical to specify the correct values for your NFS shares for the `datamover_nfs_shares` var.

Re-running this playbook will update the datamover configuration across all hosts in your inventory.

#### playbooks/deploy_dss_ai_benchmark.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to deploy the DSS AI Benchmark software.
This playbook will also create configuration files for client library and datamover, based on hosts that appear in your inventory.
Please review "Datamover Settings" under "group_vars/all.yml" if you wish to adjust the default settings of the datamover.
Uncomment vars with new values, or add them to your inventory file.

Re-running this playbook will update the datamover configuration across all hosts in your inventory.

#### playbooks/deploy_dss_software.yml

Execute this playbook to deploy DSS software to all hosts in your inventory.
This playbook will perform the following:

* Deploy, configure, and start target on all [servers]
* Deploy, configure, and start nkv-sdpk host driver to all [servers]
* Deploy, configure, and start MinIO instances to all [servers]
* Optionally deploy, configure, and start UFM to all [ufm_hosts]
* Deploy and configure datamover and client library to all [clients]

Note that core dumps are enabled on all [servers] hosts.
By default, core dumps will be compressed and stored in `/var/crash`.
Please ensure your host has enough disk space to store core dumps, if you wish to use for debugging.
This path can be changed by setting the `coredump_dir` var. see: /group_vars/all.yml

#### playbooks/format_redeploy_dss_software.yml

Execute this playbook to re-deploy DSS software to all hosts in your inventory.
This playbook is effectively identical to running "remove_dss_software", then "deploy_dss_software".
Additionally, KVSSDs (with KV firmware) will be formated cleanly.

#### playbooks/format_restart_dss_software.yml

Execute this playbook to restart DSS software to all hosts in your inventory.
This playbook is effectively identical to running "stop_dss_software", then "start_dss_software".
Additionally, KVSSDs (with KV firmware) will be formated cleanly.
SSDs (with block firmware) will be re-formated with mkfs_blobfs.

#### playbooks/redeploy_dss_ai_benchmark.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to redeploy the DSS AI Benchmark software.
This is effectively the same as executing "remove_dss_ai_benchmark" then "deploy_dss_ai_benchmark" playbooks.

#### playbooks/redeploy_dss_software.yml

Execute this playbook to redeploy DSS software to all hosts in your inventory.
This playbook is effective the same as executing "stop_dss_software", "remove_dss_software", then "deploy_dss_software".
Data present across back-end storage will persist after redeploy.

#### playbooks/remove_dss_ai_benchmark.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to remove the DSS AI Benchmark software.

#### playbooks/remove_dss_software.yml

Execute this playbook to remove DSS software to all hosts in your inventory.
This playbook is effective the same as executing "stop_dss_software" and "remove_dss_software"
Data present across back-end storage will persist after removing DSS software.

#### playbooks/remove_packet_pacing.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to remove packet pacing from servers.
This can be used to cleanup DSS AI Benchmark.

#### playbooks/remove_vlans.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to remove VLAN / IP configuration, which was previously configured with "configure_vlans" playbook.

#### playbooks/restart_dss_software.yml

Execute this playbook to restart DSS software to all hosts in your inventory.
This playbook is effective the same as executing "stop_dss_software" then "start_dss_software".

#### playbooks/start_compaction.yml

Execute this playbook to start the compaction process across all [servers].
Compaction is useful to reclaim space on backing storage devices after WRITE or DELETE operations.
Note that this playbook will wait and retry until compaction has completed across all hosts.
Compaction may take a very long time with very large datasets.
The default timeout is 12,000 seconds (200 hours)
The timeout may be changed by setting the "start_compaction_timeout" var.
For example, to start compaction with a 400 hour timeout:

    ansible-playbook -i <your_inventory> playbooks/start_compaction -e 'start_compaction_timeout=24000'

Compaction status is checked every 15 seconds by default. This value can be user-defined with the "start_compaction_delay" var.

#### playbooks/start_datamover.yml

Execute this playbook to start the datamover accross all [clients].

By default, "start_datamover" will execute a PUT operation, uploading all files from your configured NFS shares to the MinIO object store.
Also by default, compaction will run once PUT is complete.
Please review the "Datamover Settings" section of "group_vars/all.yml"
It is critical to set the "datamover_nfs_shares" to match your environment.
IPV4, IPV6, or resolvable hostnames are accepted for the "ip" key.

Additional operations supported by "start_datamover: PUT, GET, DEL, LIST, TEST

* PUT: Upload files from NFS shares to object store
* GET: Download files from object store to a shared mountpoint on all [clients]
* LIST: List objects on object store. Produces a count of objects on object store, and saves a list of objects to a default location.
* DEL: Delete all objects on object store, previously uploaded by datamover.
* TEST: Perform a checksum validation test of all objects on object store, compared to files on NFS shares.

This playbook has a number of user-definable variables that can be set from the command line to run the operation you choose:

* datamover_operation: PUT
* datamover_dryrun: false
* datamover_compaction: true
* datamover_prefix: ''
* datamover_get_path: "{{ ansible_env.HOME }}/datamover"

For example, to execute datamover GET operation to a writable, shared mount point across all [clients]:

    ansible-playbook -i <your_inventory> playbooks/start_datamover -e 'datamover_operation=GET,datamover_get_path=/path/to/share/'

For additional documentation, please consult the datamover README.md file, located on all [clients]:

> /usr/dss/nkv-datamover/README.md

#### playbooks/start_dss_ai_benchmark.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Execute this playbook to start the DSS AI Benchmark software.
This playbook will also create configuration files for client library and datamover, based on hosts that appear in your inventory.
Please review "Datamover Settings" under "group_vars/all.yml" if you wish to adjust the default settings of the datamover.
Uncomment vars with new values, or add them to your inventory file.

#### playbooks/start_dss_software.yml

Execute this playbook to start DSS software to all hosts in your inventory.
This playbook is idempotent, and will only start DSS processes if they are not already running.

#### playbooks/stop_dss_software.yml

Execute this playbook to stop DSS software on all hosts in your inventory.
This playbook is idempotent, and will only stop DSS processes if they are not already stopped.
The following actions will be performed on all servers:

1. Stop MinIO
2. Unmount NVMeOF targets / remove kernel driver
3. Stop Target

Note: Disks will remain in user space (SPDK)

#### playbooks/stop_reset_dss_software.yml

Execute this playbook to stop DSS software to all hosts in your inventory.
Additionally, once DSS software is stopped, disks will be returned back to kernel mode (SPDK reset).

#### playbooks/support_bundle.yml

Execute this playbook to collect a basic support bundle on all hosts.
A support bundle will be generated, including any coredumps found (by default, under /var/crash/),
as well as the following the following logs:

* target logs
* MinIO logs
* dmesg logs

The support bundle will be downloaded to a local path, defined by `local_coredump_dir` var.
For example, to download support bundles from all hosts to your local home directory:

    ansible-playbook -i <your_inventory> playbooks/support_bundle.yml -e 'local_coredump_dir=~/tess_support/'

By default, a support bundle will always be downloaded. To only generate a support bundle if a coredump is detected,
set the `coredump_only` var to `true`. Example:

    ansible-playbook -i <your_inventory> playbooks/support_bundle.yml -e 'local_coredump_dir=~/tess_support/,coredump_only=true'

#### playbooks/test_network.yml

Execute this playbook to perform a basic set of network tests across all hosts.
All hosts will ping each other across TCP IP's, and RoCEv2 IPs.
An ib_read_bw test will then run, and assert that measured throughput is at least 90% of link speed.

#### playbooks/test_nkv_test_cli.yml

NOTE: For internal Samsung / DSS use! Unsupported!

Perform a basic nkv_test_cli test and report observed bandwidth.

#### playbooks/upgrade_dss_software.yml

Execute this playbook to upgrade DSS software to all hosts in your inventory.
This playbook is equivelent to executing "stop_dss_software" then "deploy_dss_software"
Note that software will only be upgraded if new binaries are placed under the "artifacts" directory.

#### playbooks/upgrade_kvssd_firmware.yml

NOTE: For internal Samsung / DSS use! Unsupported!

This playbook can be used to upgrade the firmware of PM983 SSDs. All other models not supported.
In order to upgrade firmware, a valid firmware binary must be copied to the "artifacts" directory.
Then the "target_fw_version" must be commented out in all vars / defaults files.

## Testing DSS Software Stack

### Wasabi Benchmark Program (s3-benchmark)

Wasabi Benchmark (s3-benchmark) is a performance testing tool that can validate performance of standard S3 operations (PUT, GET, and DELETE) on the MinIO object store.
It is automatically installed by Ansible to `/usr/dss/nkv-minio/s3-benchmark` on each host in the [servers] and [clients] groups.

For complete documentation please see <https://github.com/wasabi-tech/s3-benchmark>.

#### s3-benchmark Help

Example s3-benchmark help:

    Wasabi benchmark program v2.0
    Usage of myflag:
    -a string
        Access key
    -b string
        Bucket for testing (default "wasabi-benchmark-bucket")
    -c int
        Number of object per thread written earlier
    -d int
        Duration of each test in seconds (default 60)
    -l int
        Number of times to repeat test (default 1)
    -n int
        Number of IOS per thread to run
    -o int
        Type of op, 1 = put, 2 = get, 3 = del
    -p string
        Key prefix to be added during key generation (default "s3-bench-minio")
    -r string
        Region for testing (default "us-east-1")
    -s string
        Secret key
    -t int
        Number of threads to run (default 1)
    -u string
        URL for host with method prefix (default "http://s3.wasabisys.com")
    -z string
    Size of objects in bytes with postfix K, M, and G (default "1M")

#### Example s3-benchmark Test

Below is an example s3-benchmark run using 100 threads, 100 objects per thread, and 1MB object size (100 GB total).
The benchmark reports throughput and operations per second for each operation (PUT, GET, and DELETE).
Results are written to `benchmark.log` in the working directory where the s3-benchmark command was invoked.

Note: Do not exceed 50% of the total subsystem capacity.
Note: Do not exceed 256 threads in VM Environment.
Note: After writing data in the storage and before reading the data, it is necessary to run compaction command.
Compaction reduces the dataset footprint on back-end storage and ensures optimal read (GET) performance.

##### Put Data to a MinIO endpoint (from a host in [clients] or [servers] group)

Command:

    [ansible@msl-ssg-vm01 ~]$ /usr/dss/nkv-minio/s3-benchmark -a minio -s minio123 -b testbucket -u http://192.168.200.1:9000 -t 100 -z 1M -n 100 -o 1

Output:

    Wasabi benchmark program v2.0
    Parameters: url=http://192.168.200.1:9000, bucket=testbucket, region=us-east-1, duration=60, threads=100, num_ios=100, op_type=1, loops=1, size=1M
    Loop 1: PUT time 40.0 secs, objects = 10000, speed = 250.2MB/sec, 250.2 operations/sec. Slowdowns = 0

##### Run Compaction (from Ansible host)

Command:

    ansible-playbook -i your_inventory playbooks/start_compaction.yml

##### Get Data (from a host in [clients] or [servers] group)

Command:

    [ansible@msl-ssg-vm01 ~]$ /usr/dss/nkv-minio/s3-benchmark -a minio -s minio123 -b testbucket -u http://192.168.200.1:9000 -t 100 -z 1M -n 100 -o 2

Output:

    Wasabi benchmark program v2.0
    Parameters: url=http://192.168.200.1:9000, bucket=testbucket, region=us-east-1, duration=60, threads=100, num_ios=100, op_type=2, loops=1, size=1M
    Loop 1: GET time 18.7 secs, objects = 10000, speed = 536MB/sec, 536.0 operations/sec. Slowdowns = 0

##### Delete Data (from a host in [clients] or [servers] group)

Command:

    [ansible@msl-ssg-vm01 ~]$ /usr/dss/nkv-minio/s3-benchmark -a minio -s minio123 -b testbucket -u http://192.168.200.1:9000 -t 100 -z 1M -n 100 -o 3

Output:

    Wasabi benchmark program v2.0
    Parameters: url=http://192.168.200.1:9000, bucket=testbucket, region=us-east-1, duration=60, threads=100, num_ios=100, op_type=3, loops=1, size=1M
    Loop 1: DELETE time 22.5 secs, 445.2 deletes/sec. Slowdowns = 0

### MinIO Client

The MinIO Client (mc) is automatically deployed to each host in your cluster at `/usr/dss/nkv-minio/mc`.

Each host in the [clients] group is configured with an alias to a single MinIO endpoint in your cluster, named `autominio`.
The default client alias can be user-defined by specifying the `minio_mc_alias` var. See: `/group_vars/all.yml`.
To view the configuration details, use:

    /usr/dss/nkv-minio/mc config host list autominio

Each host in the [servers] group is configured with an alias to each MinIO endpoint hosted on that host.
These local endpoints are prefixed with `local_`.
To view the configuration details, use:

    /usr/dss/nkv-minio/mc config host list

Example local mc aliases:

    local_msl_ssg_vm01_tcp_0
      URL       : http://msl-ssg-vm01-tcp-0:9000
      AccessKey : minio
      SecretKey : minio123
      API       : s3v4
      Lookup    : auto

    local_msl_ssg_vm1_v6_msl_lab
      URL       : http://msl-ssg-vm1-v6.msl.lab:9000
      AccessKey : minio
      SecretKey : minio123
      API       : s3v4
      Lookup    : auto

To see full MinIO Client documentation, use:

    /usr/dss/nkv-minio/mc --help

MinIO help example:

    COMMANDS:
    ls       list buckets and objects
    mb       make a bucket
    rb       remove a bucket
    cp       copy objects
    mirror   synchronize object(s) to a remote site
    cat      display object contents
    head     display first 'n' lines of an object
    pipe     stream STDIN to an object
    share    generate URL for temporary access to an object
    find     search for objects
    sql      run sql queries on objects
    stat     show object metadata
    tree     list buckets and objects in a tree format
    du       summarize disk usage folder prefixes recursively
    diff     list differences in object name, size, and date between two buckets
    rm       remove objects
    event    configure object notifications
    watch    listen for object notification events
    policy   manage anonymous access to buckets and objects
    admin    manage MinIO servers
    session  resume interrupted operations
    config   configure MinIO client
    update   update mc to latest release
    version  show version info

    GLOBAL FLAGS:
    --autocompletion              install auto-completion for your shell
    --config-dir value, -C value  path to configuration folder (default: "/opt/ansible/.mc")
    --quiet, -q                   disable progress bar display
    --no-color                    disable color theme
    --json                        enable JSON formatted output
    --debug                       enable debug output
    --insecure                    disable SSL certificate verification
    --help, -h                    show help
    --version, -v                 print the version
