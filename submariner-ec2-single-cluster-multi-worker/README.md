# Kubernetes/Submariner Multi-Cluster Ansible Deployment for AWS EC2

TLDR; the topology for this setup is a single-cluster deployment, broker node with as many worker nodes as specified.

This repo fully automates the deployment of Kubernetes on EC2 and can scale out dynamically to however
many standalone cluster nodes as desired. The playbooks use the latest changes to the `amazon.aws.ec2_instance` module.

Currently, the Kubernetes distribution is [K3s](https://github.com/k3s-io/k3s) as this is focused on datapath
performance testing, so the lighter the weight the better for our needs. Once the EC2 nodes are up and running, Skupper is 
then provisioned for multi-cluster service exports. We will be adding Microshift as an alternative lightweight 
distribution as soon as this PR merges [Allow MicroShift to join new worker nodes](https://github.com/redhat-et/microshift/pull/471).

### Quickstart for single-cluster multi-submariner gateway, multi-worker

- Edit `ansible.cfg` adding in the relevant information. For the Axon group, `private_key_file` in
  `ansible.cfg` in vars.yaml is the main field to change is the location of the perf testing private key.
  Put the private key in the same directory as ansible.cfg or specify the path and set the key permissions
  .e.g. `chmod 0400 axon-perf-testing.pem`
- One broker node and one secondary gatway nodes are spawned in the same cluster.
- In `vars.yml` the `node_count:` key specifies how many worker nodes are spawned.
- The broker node is also the k8s master node for the cluster.
- broker-info.shim and k8s server tokens are stored in the base directory.

```sh
ansible-vault create credentials.yml
```

- Setup the EC2 nodes by running the playbook and specify the host root password when asked for `BECOME PASSWORD` and then supply your
  vault password when asked.

This will result in:
- 1x broker node also running as the k8s master
- 1x secondary gateway node running as a k8s worker
- (n)x k8s worker nodes, n=node_count value specified in `vars.yml`

```sh
ansible-playbook --ask-vault-pass setup-ec2.yml
```

- Next deploy the k8s/submariner playbooks:
- TODO: this last step is still a WIP. Kubeconfig needs to be copied down from the master note to the submariner secondary gateway in order to run submariner pods.

```sh
ansible-playbook setup-k8s.yml
```

### Setup

- This setup installs the specified number of k8s clusters, installs submariner and joins them to the broker.
- More details on pre-requisites of the setup can be found at [Kubernetes Multi-Node Ansible Deployment for AWS EC2](https://github.com/nerdalert/aws-ansible-kubernetes/blob/main/README.md).

```sh
# The following will open an editor
ansible-vault create credentials.yml

# Paste the following in the opened editor and save the file
access_key: <add_access_key_here>
secret_key: <add_secret_key_here>
```

- Adjust the `ansible.cfg` file in the base directory to reflect your environment.
- The main one that needs to be edited for your environment will be the path to the
  `private_key_file.pem` entry that is associated with your AWS key specified in `vars.yml`.
  You can also simply copy the key to the base directory and leave off the path.
- In some environments you won't need to enable `become_ask_pass` but adding it to be as
  agnostic as possible to all installations.

```yaml
[defaults]
# this is an default inventory location, user can change it accordingly
host_key_checking = false
deprecation_warnings = false
ask_pass = false
stdout_callback = yaml
remote_user = fedora
# defaults to the base directory in the project
inventory = ip.txt
# create .pem private_key_file and provide location
private_key_file = <aws_private_key_name>.pem

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = true
```

- Next set the environmentals for your AWS EC2 details in `env.yaml` located in the base
  of the project. Here are some example values.

```yaml
aws_region: us-east-1                 # AWS region
vpc_id: vpc-xxxxxxxx                  # VPC id from your aws account
aws_subnet: subnet-xxxxxxxx           # VPC subnet id from your aws account
aws_image_id: ami-08b4ee602f76bff79   # Fedora 35 (this can be changed to most any Linux distro)
aws_key_name: <ec2-keypair_name>      # the key pair on your aws account to use
aws_instance_type: t2.xlarge          # t2.micro is free tier eligible, but you can use any type to scale up, more examples [t2.large, t2.xlarge, t2.2xlarge]
ansible_user: fedora                  # this is the default user ID for your AMI image. Example, AWS AMI is ec2-user etc
node_count: 14                        # the number of cluster nodes you want to deploy
secgroup_name: perf_testing_secgroup  # the security group name can be an existing group or else it will be created by the playbook
security_group_description: "Security Group for Perf/Scale testing allowing ssh ingress"
inventory_location: ip.txt            # leaving this as is will use the ip.txt file in the base directory
submariner_cluster_cidr: 10.42.0.0/24
submariner_external_cidr: 172.31.16.0/20
submariner_cable_driver: vxlan        # [vxlan, ipsec]
```

### Run the installation

- Once your `env.yaml` is setup for your EC2 environment, you are ready to run the playbook to deploy the EC2 instances.
  This will create the VMs for the deployment (add -vv for verbose output).

```sh
$ ansible-playbook --ask-vault-pass setup-ec2.yml
# host being run from su password (may not be required in some setups, can disable in ansible.cfg)
BECOME password:
# password when you created the ansible vault
Vault password:
```

- After that run is complete, you can always ping the nodes to verify connectivity by running the following from the base directory:

```sh
ansible all -m ping
```

```sh
ansible-playbook setup-ec2.yml
```

- You can double check connectivity to the new nodes with:

```sh
# This pings all of the hosts in your ip.txt file 
ansible all -m ping
```

- Once your nodes are running, deploy K3s Kubernetes to the nodes listed in your inventory file `ip.txt` by running the playbooks in `setup-kubernetes.yml`

```
ansible-playbook setup-k8s.yml
```

Assuming that runs with no issues, your k8s/submariner deployment is up and running.
- The `broker-info.subm` file used with all joins is copied to the local machine  
from the initial broker node installation and then copied to all cluster nodes for 
their join.
- The kube server join token is copied to the local directory if you want to attach
worker nodes to any of the clusters.

### Verify the Kubernetes/Submariner Deployment

You can find the addresses for the installed nodes in `ip.txt`, for example:

```yaml
[brokerNode]
  54.147.129.101 ansible_user=fedora ansible_connection=ssh

 [workerNode]
  34.228.144.25 ansible_user=fedora ansible_connection=ssh
  54.226.11.155 ansible_user=fedora ansible_connection=ssh
  54.174.214.75 ansible_user=fedora ansible_connection=ssh
  54.146.23.141 ansible_user=fedora ansible_connection=ssh
  54.226.202.170 ansible_user=fedora ansible_connection=ssh
  50.17.32.20 ansible_user=fedora ansible_connection=ssh
  34.227.78.13 ansible_user=fedora ansible_connection=ssh
  54.157.241.103 ansible_user=fedora ansible_connection=ssh
  34.227.195.1 ansible_user=fedora ansible_connection=ssh
  52.23.223.51 ansible_user=fedora ansible_connection=ssh
  34.235.155.89 ansible_user=fedora ansible_connection=ssh
  35.175.195.234 ansible_user=fedora ansible_connection=ssh
  54.242.206.254 ansible_user=fedora ansible_connection=ssh
  54.226.40.73 ansible_user=fedora ansible_connection=ssh
...

```

- Connect and verify the Submariner installation:

```
# ssh to the master node. you can look in ip.txt for the ip address
ssh -i ./<aws-private-key>.pem  fedora@<any_cluster_address_from_inventory>

# export the kube config
export KUBECONFIG=~/.kube/config

# view submariner status pods
subctl show all
Cluster "default"
 ✓ Detecting broker(s)
NAMESPACE                NAME                     COMPONENTS
submariner-k8s-broker    submariner-broker        service-discovery, connectivity

 ✓ Showing Connections
GATEWAY                CLUSTER                  REMOTE IP      NAT  CABLE DRIVER  SUBNETS        STATUS     RTT avg.
172-31-28-249-cluster  172-31-28-249-cluster-c  172.31.28.249  no   vxlan         242.4.0.0/16   connected
172-31-21-152-cluster  172-31-21-152-cluster-c  172.31.21.152  no   vxlan         242.1.0.0/16   connected
172-31-20-57-cluster   172-31-20-57-cluster-cl  172.31.20.57   no   vxlan         242.2.0.0/16   connected
172-31-22-202-cluster  172-31-22-202-cluster-c  172.31.22.202  no   vxlan         242.3.0.0/16   connected
172-31-19-47-cluster   172-31-19-47-cluster-cl  172.31.19.47   no   vxlan         242.5.0.0/16   connected
172-31-20-96-cluster   172-31-20-96-cluster-cl  172.31.20.96   no   vxlan         242.6.0.0/16   connected
172-31-21-159-cluster  172-31-21-159-cluster-c  172.31.21.159  no   vxlan         242.7.0.0/16   connected
172-31-28-119-cluster  172-31-28-119-cluster-c  172.31.28.119  no   vxlan         242.8.0.0/16   connected
172-31-30-210-cluster  172-31-30-210-cluster-c  172.31.30.210  no   vxlan         242.9.0.0/16   connected
172-31-18-60-cluster   172-31-18-60-cluster-cl  172.31.18.60   no   vxlan         242.10.0.0/16  connected
172-31-19-200-cluster  172-31-19-200-cluster-c  172.31.19.200  no   vxlan         242.12.0.0/16  connected
172-31-28-66-cluster   172-31-28-66-cluster-cl  172.31.28.66   no   vxlan         242.11.0.0/16  connected
172-31-16-217-cluster  172-31-16-217-cluster-c  172.31.16.217  no   vxlan         242.14.0.0/16  connected
172-31-29-109-cluster  172-31-29-109-cluster-c  172.31.29.109  no   vxlan         242.13.0.0/16  connected
```

You are all set from there, feel free to leave feedback, open issues and/or PRs!
