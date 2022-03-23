# Kubernetes/Skupper Multi-Cluster Ansible Deployment for EC2

This repo fully automates the deployment of Kubernetes on EC2 and can scale out dynamically to however
many standalone cluster nodes as desired. Once the EC2 nodes are up and running, Skupper is then provisioned
for multi-cluster service exports. The playbooks use the latest changes to the `amazon.aws.ec2_instance` module.

Currently, the Kubernetes distribution is [K3s](https://github.com/k3s-io/k3s) as this is focused on datapath
performance testing, so the lighter the weight the better for our needs. We will be adding Microshift as an
alternative lightweight distribution as soon as this PR merges [Allow MicroShift to join new worker nodes](https://github.com/redhat-et/microshift/pull/471).


### Setup

- This setup installs the specified number of k8s clusters, installs [Skupper](https://github.com/skupperproject) and joins them to the broker.
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

Assuming that runs with no issues, your k8s/skupper deployment is up and running.
- Each hub/spoke link key is stored in `skupper-keys-tmp/`. The key for each spoke node
is copied onto the spoke and provisioned.
- The kube server join token is copied to the local directory if you want to attach
  worker nodes to any of the clusters.

### Verify the Kubernetes/Skupper Deployment

You can find the addresses for the installed nodes in `ip.txt`, for example:

```yaml
[hubNode]
54.85.51.200 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-hub-node

[spokeNode]
3.84.187.196 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-9
3.89.113.154 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-8
54.227.184.57 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-7
54.234.180.4 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-6
54.81.95.253 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-5
3.95.59.110 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-4
35.175.174.232 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-3
54.81.219.90 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-2
3.95.241.80 ansible_user=fedora ansible_connection=ssh ec2_name=skupper-spoke-node-1
```

- Connect and verify the Skupper installation:

```
# ssh to the master node. you can look in ip.txt for the ip address
ssh -i ./<aws-private-key>.pem  fedora@<hub_ip_from_inventory>

# export the kube config
export KUBECONFIG=~/.kube/config

# view skupper status
$ skupper status
Skupper is enabled for namespace "skupper-hub-node" in interior mode. It is connected to 9 other sites. It has no exposed services.
The site console url is:  https://172.31.26.203:8080
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
```

You are all set from there, feel free to leave feedback, open issues and/or PRs!


