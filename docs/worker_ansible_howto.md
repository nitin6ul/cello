Cello ansible agent how-to
==========================

Cello ansible agent is a set of ansible playbooks which allows developers and
fabric network operators to stand up fabric network very quickly in a set of
virtual or physical servers in cloud or lab.

The following is the list of the general steps you can follow to use the
ansible agent to stand up your own fabric network::

1. [Setup ansible controller](#setup-ansible-controller)
2. [Create configuration files](#create-configuration-files)
3. [Provision the servers](#provision-the-servers)
4. [Initialize the servers](#initialize-the-servers)
5. [Setup fabric network](#setup-fabric-network)
6. [Verify fabric network](#verify-fabric-network)

This document also provides the following sections to help you use
Cello ansible agent:

[Cloud configuration file details](#cloud-configuration-file-details)<br/>
[Fabric configuration file details](#fabric-configuration-file-details)<br/>
[Running an ansible playbook](#running-an-ansible-playbook)<br/>
[Use ssh-agent to help Ansible](#ssh-agent-to-help-ansible)<br/>
[Convenient configurations and commands](#ccac)<br/>
[Use the existing servers](#use-the-existing-servers)<br/>
[Security rule references](#srrwy)<br/>

# <a name="setup-ansible-controller"></a>Setup ansible controller

Ansible playbooks run on Ansible controller. Ansible controller is basically
a computer which has ansible and dependent software installed. It can be your
own laptop, virtual or physical machine. Here are four steps to create an
ansible controller to run cello ansible agent.

1. [Install ansible](#install-ansible)
2. [Generate a ssh key pair](#generate-a-ssh-key-pair)
3. [Clone ansible agent code](#clone-ansible-agent-code)
4. [Install ansible cloud module](#install-ansible-cloud-module)


## <a name="install-ansible"></a>Install ansible

To run ansible playbooks ansible controller can be any machine, as long as
you can install ansible 2.3.0.0 or above onto, it can be a small virtualbox
VM, your own laptop or an existing VM which runs in your own cloud.

Please follow offical [ansible installation instructions](http://docs.ansible.com/ansible/latest/intro_installation.html)
from ansible web site to install ansible. If you like to start quickly and
have a clean Ubuntu 16.04 system, here are the commands to install ansible::

        sudo apt-get update
        sudo apt-get install python-pip -y
        sudo pip install 'ansible>=2.3.0.0'

Once it is installed, you should run following command to check its version::

        ansible --version

The results should show something at or above 2.3.0.0. It is entirely possible
that you may have to install additional software that ansible depends on in
some older versions of operating systems.

## <a name="generate-a-ssh-key-pair"></a>Generate a ssh key pair

Ansible heavily rely on ssh to work with virtual or physical servers. To
establish a ssh connection with the servers, you will need to have a pair of
ssh key, you may choose to use your own existing one, but it is highly
recommended to create one for test purposes. Here are the steps to generate
a key pair::

        mkdir -p ~/.ssh && cd ~/.ssh && ssh-keygen -t rsa -f fd -P ""

The above procedure will create a non passphrase ssh key pair in your ~/.ssh
directory. The name of the files will be fd and fd.pub. In default ansible
agent default configuration files, the ssh key is assumed at that location
with the name fd and fd.pub. If you use different name, then you will need
to change the configuration with your own ssh key name.

When you work with have a cloud such as OpenStack, AWS, the ssh public key
you generated here will be automatically injected into the severs in
[Provision the servers](#provision-the-servers) step. If your servers were
not provisioned by ansible agent, you will have to manually inject the public
key into each machine. You will need to place the ssh public key in the file
named ~/.ssh/authorized_keys in the user account which you will use to log
into the server. You will need to do this for each server.

Once you have the ssh keys, you will need to start up the ssh agent to so
that the following steps will not keep asking you. Do execute the following
two commands:

        eval $(ssh-agent -s) && ssh-add ~/.ssh/fd

## <a name="clone-ansible-agent-code"></a>Clone ansible agent code

Ansible agent is part of the hyperledger cello project. To run its playbooks,
you will need to extract the code. Follow the follow procedure to get code::

       cd ~
       git clone --depth=1 https://gerrit.hyperledger.org/r/cello

This command will extract the latest cello code. Ansible agent is at the
following directory::

       ~/cello/src/agent/ansible

from this point on, you should stay in that directory for all command
executions. This directory is also referred to root agent directory. For
convenience, you can execute the following command to help you get back to
the root easily

       export AAROOT=~/cello/src/agent/ansible
       cd $AAROOT

## <a name="install-ansible-cloud-module"></a>Install ansible cloud module

When deals with cloud such as AWS, Azure or OpenStack, Ansible requires
some libraries to access these clouds. These libraries are normally called
[ansible cloud modules](http://docs.ansible.com/ansible/latest/list_of_cloud_modules.html).
You will find a great details regarding these modules. When you work with
a specific cloud, you will need only to install the cloud modules for that
cloud. Here are the steps to install ansible modules for AWS, Azure and
OpenStack::

        AWS:
        sudo pip install boto boto3

        Azure:
        sudo pip install azure

        OpenStack:
        sudo pip install shade

These modules are needed in the [Provision the servers](#provision-the-servers)
step, if you are not using ansible agent to provision your servers, you do
not need any of these modules.

# <a name="create-configuration-files"></a>Create configuration files

Ansible agent relies on two configuration files to work and stand up your
fabric network. One is called cloud configuration file, the other is called
fabric configuration file.

Cloud configuration file is used to work with the cloud such as AWS, Azure
and OpenStack to create virtaul machines, create security rules for network,
inject ssh key, eventually delete virtual machines when you decide to destroy
everything you created. Ansible agent provides the following sample
cloud configuration files::

        $AAROOT/vars/aws.yml
        $AAROOT/vars/azure.yml
        $AAROOT/vars/os.yml

Fabric configuration file is used to create a fabric network according to
your need. This file will let you specify what release of fabric you want to
use, what organizations to create, how many peers or orderers in an
organization, how many kafka and zookeeper nodes you like to have and how
you want to layout your fabric network on the servers you have. Ansible
agent provides the following sample fabric configuration files::

        $AAROOT/vars/bc1st.yml
        $AAROOT/vars/bc2nd.yml
        $AAROOT/vars/vb1st.yml

It is these two files ultimately control how your fabric network look like.
You are expected to use the listed files above as examples to create your
own cloud configuration and fabric configuration files. You should make a
copy from each type of the examples files, then make changes based on your
own desire and credentials to run ansible playbook to produce your fabric
network. The copy of your own configuration files should also be in the same
directory where these example files reside. Otherwise, ansible agent will
not be able to find them. You can certainly make changes directly to one of
these files and use them.

For an easier discussion, assume you have created your own cloud configuration
file namde `mycloud.yml` and your own fabric configuration file named
`myfabric.yml`. The content of each the two files must follow the respective
sample file. Since both files are relatively large, have a good understanding
of what each field means in both files is absolutely critical. Please refer
to the following two sections for details on each field in the two files.

[Cloud configuration file details](#cloud-configuration-file-details)<br/>
[Fabric configuration file details](#fabric-configuration-file-details)

# <a name="provision-the-servers"></a>Provision the servers

This step is to provision a set of virtual servers from a cloud.

        ansible-playbook -e "mode=apply env=mycloud cloud_type=os" provcluster.yml

The above command will provision (prov is short for provision) a cluster of
virtual machines on your OpenStack cloud the environment defined in
vars/mycloud.yml file. value apply for parameter mode indicates you like to
create resources, value os for parameter cloud_type indicates that you are
using OpenStack cloud, value mycloud for parameter env indicates you are
using vars/mycloud.yml file. The possible values for mode are `apply` and
`destroy`, the possible values for cloud_type are `os`, `aws` and `azure` at
present.

This step produces a set of servers in your cloud and an ansible host file
named runhosts in this directory on your ansible controller::

        $AAROOT/run

If you are working with servers already exist, you will need to follow
the section [Use the existing servers](#use-the-existing-servers) to continue
setting up your fabric network.

To remove everything this step created, run the following command::

        ansible-playbook -e "mode=destroy env=mycloud cloud_type=os" provcluster.yml

# <a name="initialize-the-servers"></a>Initialize the servers

This step will install all necessary software packages, setup an overlay
network, dns services and registrator services on the machines created in
previous step::

        ansible-playbook -i run/runhosts -e "mode=apply env=mycloud env_type=flanneld" initcluster.yml

The parameter env is same as in previous step. The parameter env_type
indicates what communication environment you like to setup. The possible
values for this parameter are `flanneld` and `k8s`. Value `flanneld` is to
setup a docker swarm like environment. Value `k8s` is to setup kuberenetes
environment.

To remove everything this step created, run the following command::

        ansible-playbook -i run/runhosts -e "mode=destroy env=mycloud env_type=flanneld" initcluster.yml

# <a name="setup-fabric-network"></a>Setup the fabric network

This step will build or download from a docker repository fabric binaries
and docker images, create certificates, and eventually run various fabric
components such as peer, orderer, kafka, zookeeper, fabric ca on the
environment produced in previous steps::

        ansible-playbook -i run/runhosts -e "mode=apply env=myfabric deploy_type=compose" setupfabric.yml

The env value in the command indicates which fabric network configuration to
use. The meaning of this parameter is a bit different comparing to the
previous commands. The parameter deploy_type indicates if like to use docker
compose to deploy or use k8s to deploy.

To remove everything this step created, run the following command::

        ansible-playbook -i run/runhosts -e "mode=destroy env=myfabric deploy_type=compose" setupfabric.yml

# <a name="verify-fabric-network"></a>Verify fabric network

If all previous steps run without any errors, you should have a running
status of each container::

        ansible-playbook -i run/runhosts -e "mode=verify env=bc1st" verify.yml

The above command should acess all the servers and display all the container
status in your fabric network. If these containers did not exit, then you know
you have successfully deployed your own fabric network.

# Useful tips for running ansible agent

## <a name="cloud-configuration-file-details"></a>Cloud configuration file details

Cloud configuration file is used by ansible agent to work with cloud. It is
very important to make every field in the file accurate according to your
own cloud. Most of the information in this type of the files should be
provided by your cloud providers. If you are not 100% sure what value a field
should have, it would be a good idea to use the corresponding value in the
sample cloud configuration file. The following section describes what each
field means. Please use [vars/os.yml](https://github.com/hyperledger/cello/blob/master/src/agent/ansible/vars/os.yml)
as a reference and see example values for these fields::

```
auth: Authorization fields for a cloud
auth_url: url for cloud Authorization
username: User name to log in to the cloud account
password: Password for the user of the cloud account
project_name: Name of the project of your cloud, specific to OpenStack
domain: The domain of the project, specific to OpenStack
cluster: This section defines how virtual machines should be created
target_os: The operating system we are targeting, it has to be `ubuntu`
    at present
image_name: Cloud image name to be used to create virtual machines
region_name: Region name for VMs to reside in, leave blank if unsure
ssh_user: The user id to be used to access the virtual machines via ssh
availability_zone: The availability zone, leave blank to use default
validate_certs: When access cloud, should the certs to be validated?
    Set to false if your cloud use self signed certificate
private_net_name: The private network name from your cloud account on
    which your VMs should be created
flavor_name: Flavor name to create your virtual machine
public_key_file: The public ssh key file used to connect to servers,
    use absolute path
private_key_file: The private ssh key file, use absolute path,
    public_key_file and this key file should make a pair
node_ip: Use public ip or private ip to access each server, only possible
    value are `public_ip` and `private_ip`
assign_public_ip: Should each VM be allocated public IP address or not, true
    or false, default is true
container_network: This section defines overlay network, you should not
    change this unless you absolutely know what you are doing
Network: Overlay network address space, should be always in cidr notion,
    such as 172.16.0.0/16
SubnetLen: The bit length for subnets, it should be 24 normally, do not
    change it unless you absolutely know what you are doing
SubnetMin: minimum subnet
SubnetMax: maximum subnet
Backend: backend for flanneld setup
Type: the type for flanneld setup
Port: the port to use
service_ip_range: when use k8s, this defines service ip range
dns_service_ip: dns service ip address for k8s
node_ips: a list of public IP addresses if you like the VMs to be accessible
    and using preallocated IP addresses
name_prefix: VM name prefix when create new VMs, this combines with
    stack_size to make VM names. These names will be used in fabric
    configuration For example,if your prefix is fabric, and stack
    size is 3, then you will have 3 VMs named fabric001, fabric002,
    fabric003, these names will be referred as server logic names
domain: domain name to use when create fabric network nodes.
stack_size: how many VMs to be created
etcdnodes: which nodes you like the etcd to be set up on. only needed
    for k8s, should be a list of logic name like fabric001, fabric002,
    fabric003
builders: which VM to be used to build fabric binaries. should be only one
    machine, use logic name like fabric001, fabric002, etc.
flannel_repo: The url point to the flanneld tar gz file
etcd_repo: The url point to the etcd tar gz file
k8s_repo: The url point to the k8s binary root directory
go_repo: The url point to the go lang tar gz file
volume_size: when create VMs the size of the volume
block_device_name: block device name when create volume on OpenStack cloud
    fabric network, to verify that, you can run the following command to see
```

## <a name="fabric-configuration-file-details"></a>Fabric configuration file details

Fabric configuration file defines how your fabric network should look like,
how many servers you will use and how many organizations you would like to
create and how many peers, orderers should each organization has, how many
kafka and zookeeper containers you like to setup. what names you like to
give to organizations, peers, orderers etc. It is this file that controls
ultimately the topology of your fabric network. Get a good understanding
of this file is essential to create a fabric network according to your
need. Please use [vars/bc1st.yml](https://github.com/hyperledger/cello/blob/master/src/agent/ansible/vars/bc1st.yml)
as a reference and see example values for these fields::

```
GIT_URL: hyperledger fabric git project url. should be always
    "http://gerrit.hyperledger.org/r/fabric"
GERRIT_REFSPEC: ref spec when build a specifc patch set. for example, it
    can be "refs/tags/v1.0.5"
fabric: This section define hyperledger fabric network layout
ssh_user: The user name to be used to log in to the remote servers
peer_db: The peer database type, possible values are CouchDB and leveldb
tls: Should this deployment use tls, default is false,
network: This section defines the layout of the fabric network
fabric001: This defines fabric containers running on the node named
    fabric001, each virtual or physical machine should have a section
    like this.
cas: list of the fabric certificate authority for an organization,
    the name of each ca should be in the format of <name>.<orgname>,
    for example, ["ca1st.orga", "ca1st.orgb"]
peers: list of the peers run on this node, the format of the names
    shuold be <role>@<name>.<orgname>, for example,
    ["anchor@peer1st.orga","worker@peer2nd.orga"], this means that
    there will be two peers running on this node, they are both from
    organization named orga, one is acting as
    anchor node, the other is the worker node.
orderers: list of the orderers run on this node, the format of the
    names should be <name>.<orgname>, for example, ["orderer1st.orgc",
    "orderer2nd.orgc"], this means that there will be two orderers
    running on this node, they are both from organization named orc,
    one is named orderer1st, and the other named orderer2nd.
zookeepers: list of the zookeeper containers run on this node. The
    format for zookeeper containers are <name>, since zookeeper
    containers do not belong to any organization, their names should
    be simply a string. For example: ["zookeeper1st", "zookeeper2nd"],
    this means that there will be two zookeeper containers running on
    this node, their names are zookeeper1st and zookeeper2nd respectively.
kafkas: list of the kafka containers run on this node The format for
    kafka containers are <name>, since kafka containers do not belong
    to any organization, their name should be simply a string. For
    example, ["kafka1", "kafka2"], this means that there will be two
    kafka containers running on this node, their names are kafka1 and
    kafka2.
baseimage_tag: docker image tag for fabric-peer, fabric-orderer,
    fabric-ccenv,fabric-tools. for example, it can be "x86_64-1.1.0-alpha",
    The value of this field is very important, if this value is empty,
    that means you like to build the fabric binaries and possibly docker
    container images. This field and the repo section determins where to
    download binaries or should binaries be downloaded.
helper_tag: docker image tag for container fabric-ca, fabric-kafka,
    fabric-zookeeper, for example, it be "x86_64-1.1.0-preview"
ca: This section defines how the fabric-ca admin user id and password
admin: ca user admin name
adminpw: ca admin user password
repo: This section defines where to get the fabric docker image and
    binary tar gz file. This allows you to use a local docker repository
url: Docker image repository for fabric, for example if you are using
    docker hub, the value will be "hyperledger/", if you are using
    nexus3, the value will be "nexus3.hyperledger.org:10001/hyperledger/"
bin: The url point to the fabric binary tar gz file which contains
    configtxgen, configtxlator, cryptogen etc.
```

## <a name="running-an-ansible-playbook"></a>Running an ansible playbook

Ansible allows you to run tasks in a playbook with particular tags or skip
particular tags. For example, you can run the follow command

```
    ansible-playbook -i run/runhosts -e "mode=apply env=bc1st \
    deploy_type=compose" setupfabric.yml --tags "certsetup"
```
The above command will use the runhosts inventory file and only run tasks
or plays taged certsetup, all other plays in the play books will be
skipped.

```
    ansible-playbook -i run/runhosts -e "mode=apply env=bc1st \
    deploy_type=compose" setupfabric.yml --skip-tags "certsetup"
```
The above command will run all the tasks but the tasks/plays taged certsetup

## <a name="ssh-agent-to-help-ansible"></a>Use ssh-agent to help ansible

Since ansible access either the virtual machines that you create on a
cloud or machines that you may already have by using ssh, setting up
ssh-agent on the ansible controller is very important, without doing
this most likely, the script will fail to connect to your servers.
Follow the steps below to setting your ssh-agent on ansible controller
which should be always the machine that you run the ansible script.

1. Create a ssh key pair (only do this once)::

        ssh-keygen -t rsa -f ~/.ssh/fd

2. Run the command once in a session in which you run the ansible script::

        eval $(ssh-agent -s)
        ssh-add ~/.ssh/fd

3. For the servers created in the cloud, this step is already done for
you. For the existing servers, you will need to make sure that the fd.pub
key is in the file ~/.ssh/authorized_keys. Otherwise, the servers will
reject the ssh connection from ansible controller.

## <a name="ccac"></a>Convenient configurations and commands

At the root directory of the ansible agent, there are set of preconfigured
playbooks, they were developed as a convienent playbooks for you if you
mainly work with a particular cloud. Here are the list of these playbooks.

```
aws.yml
awsk8s.yml
os.yml
osk8s.yml
vb.yml
vbk8s.yml
```

These files were created to use coresponding cloud and fabric configuration
files. For example, aws.yml uses vars/aws.yml and vars/bc1st.yml to setup
multiple nodes fabric network on AWS cloud. awsk8s.yml uses vars/aws.yml and
vars/bc1st.yml to setup multiple node fabric network on AWS with k8s cluster.
To use these playbooks, you simply need to make small changes in the coresponding
configuration files in vars directory, then issue the following command:

To stand up a fabric network on AWS:
```
    ansible-playbook -e "mode=apply" aws.yml
```
To destroy a fabric network on AWS:
```
    ansible-playbook -e "mode=destroy" aws.yml
```

If your target environment is OpenStack, then you will be using a slightly different
commands:

```
    ansible-playbook -e "mode=apply" os.yml or osk8s.yml
    ansible-playbook -e "mode=destroy" os.yml or osk8s.yml
```

## <a name="use-the-existing-servers"></a>Use the existing servers

When you have a set of physical servers or a virtual machines already
available, you can still use the ansible agent to stand up your fabric
network. To do that, you will need to provide what provisioning step create.

There are two things you basically need to do, one is to ensure that
your servers can be accessed via ssh, the second is to produce a runhosts
file like below, the hostnames of these servers have to form a patten using
a prefix then three digits, for example, fabric001, fabric002, fabric003.
The word fabric serves as a prefix which can be changed to any string in
the cloud configuration file. After prefix, three digits should start at
001, then all the way to the stack size. In below example, the prefix is
fabric, but you can use any string you prefer as long as it is same as
the cloud configuration file name_prefix field::

```
cloud ansible_host=127.0.0.1 ansible_python_interpreter=python
169.45.102.186 private_ip=10.0.10.246 public_ip=169.45.102.186 inter_name=fabric001
169.45.102.187 private_ip=10.0.10.247 public_ip=169.45.102.187 inter_name=fabric002
169.45.102.188 private_ip=10.0.10.248 public_ip=169.45.102.188 inter_name=fabric003

[allnodes]
169.45.102.186
169.45.102.187
169.45.102.188

[etcdnodes]
169.45.102.186
169.45.102.187
169.45.102.188

[builders]
169.45.102.186
```

The above file is a typical ansible host file. The cloud ansible_host should be your ansible
controller server, you should not change that line. All other lines in the file represent
a server, private_ip and public_ip are the concept for cloud, if your servers are not in
a cloud, then you can use the server's IP address for both private_ip and public_ip field,
but you can not remove these two fields. The inter_name is also important, you should name
the server sequentially and these names will be used in later configuration to allocate
hyperledger fabric components. Group allnodes should list all the servers other than the
ansible controller node. Group etcdnodes should list all the servers that you wish to install
etcd services on. Group builders should container just one server that you wish to use to build
hyperledger fabric artifacts such as executables and docker images.

## <a name="srrwy"></a>Security rule references when you setup fabric network on a cloud

When you work with a cloud, often it is important to open or close certain
ports for the security and communication reasons. The following port are
used by flanneld overlay network and other services of fabric network, you
will need to make sure that the ports are open. The following example assumes
that the overlay network is 10.17.0.0/16 and the docker host network is
172.31.16.0/20, you should make changes based on your network::

    Custom UDP Rule  UDP  8285              10.17.0.0/16
    Custom UDP Rule  UDP  8285              172.31.16.0/20
    SSH              TCP  22                0.0.0.0/0
    Custom TCP Rule  TCP  2000 - 60000      10.17.0.0/16
    Custom TCP Rule  TCP  2000 - 60000      172.31.16.0/20
    DNS (UDP)        UDP  53                172.31.16.0/20
    DNS (UDP)        UDP  53                10.17.0.0/16
    All ICMP - IPv4  All  N/A               0.0.0.0/0


<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">
<img alt="Creative Commons License" style="border-width:0"
src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a><br />
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/">
Creative Commons Attribution 4.0 International License</a>.