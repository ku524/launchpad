# launchpad

SpaceONE infrastructure automation provisioning code

* Based on AWS
* Implementation of Terraform and Ansible

<hr/>


## Architecture
 
### MongoDB
![MongoDB](./mongodb_arch_img.jpg)
### Backup
![MongoDB_backup](./mongodb_backup_arch.png)

## Terraform

* Guaranteed Terraform Version 0.14.2
* Separated workspace from each supported parts
* Thus, Each parts is must be executed separately

### Supported Part list
* mongodb

### Modules
* bastion
* route53
* security_group
* shard_cluster

<br/>

### How to use

#### 1. Terraform init

we recommended to use S3 Backend for state management and state locking.

`backend.tfvars` file looks like

```yaml
# AWS S3 bucket name for backend
bucket = ""

# AWS S3 object key for tfstate file
key = ""

region = ""

# support locking via DynamoDB
dynamodb_table = ""
```

Fill out `backend.tfvars` values and `terraform init` run

```sh
> terraform init --var-file=backend.tfvars
```

<br/>

#### 2. Fill out `*.auto.tfvars` for environments

Each parts included `.auto.tfvars` files for your environment properly.

For example, mongodb parts included `security_group.auto.tfvars` and `shard_cluster.auto.tfvars`.

Let's see a `security_group.auto.tfvars` for instance.
```yaml
# security_group.tfvars

region                        =   ""
vpc_id                        =   ""

mongodb_bastion_ingress_rule_admin_access_security_group_id = ""    # From Source security group ID for Administrator access
mongodb_bastion_ingress_rule_admin_access_port              = 0
mongodb_app_ingress_rule_mongodb_access_security_group_id   = ""    # From Source security group ID for Worker Nodes
```

You can fill out all of `.auto.tfvars` in each parts.

#### 2-1. Fill out SSH PEM String to access ec2 instances through SSH for Ansible if you want

Configure for Ansible in bastion instance automatically.
Fill out SSH PEM Key string to access each nodes through bastion Ansible if you want.
PEM Key string must be the same when deploy instances selected key pairs.

`spaceone/terraform/modules/ssh_pem/mongodb.pem`

```pem
# mongodb.pem
-----BEGIN RSA PRIVATE KEY-----
ENTER_YOUR_PRIVATE_KEY_STRING
-----END RSA PRIVATE KEY-----
```

if you don't set SSH Key, you must set PEM key in bastion manually for Ansible after deployment.

#### 3. Terraform plan

And then, make file for your environment.

We already add `default.auto.tfvars` for example.

`default.auto.tfvars` is simple. It looks like

```yaml
environment   = "dev"
region        = ""
```

Fill out the values in `default.auto.tfvars`.

It's time to run the `Terraform plan` !

If you run the mongodb with all of `tfvars` in mongodb parts,
```sh
> terraform plan
```

#### 4. Terraform apply

Go to build your spaceONE using launchpad !
```sh
> terraform apply
```

<br>
<hr/>


## Ansible


### Roles
* ubuntu_base
* mongodb_base
* mongod
* mongos
* config_replicaset
* data_node_replicaset
* shard_cluster
* bastion
* backup1_install_pbm
* backup2_db_backup_role
* backup3_shardwide_setting
* backup4_start_agent

### How to use

#### 1. Access Bastion Instance

#### 2. Access Test (dynamic inventory)

We will use Ansible dynamic inventory. 
Configuration for using Ansible dynamic inventory is already filled except one thing - region.

`spaceone/ansible/inventory/aws_ec2.yml`
```yaml
# aws_ec2.yml
plugin: aws_ec2

regions:
  - us-west-1 # region where mongodb place
...
```
Please check do `ansible-inventory` command.

```sh
> ansible-inventory --graph -i inventory/
```

The result should look like this
```
@all:
  |--@aws_ec2:
  |  |--mongodb-bastion
  |  |--mongodb-cfg1
  |  |--mongodb-cfg2
  |  |--mongodb-cfg3
...
```

#### 3. Fill out Variables in group variable file - mongodb.yml

`spaceone/ansible/inventory/group_vars/mongodb.yml`

We will run ansible playbook `mongodb.yml` only. This playbook is included variety roles for configuration MongoDB.
You must set variables in this playbook for use.

#### 4. Run playbook!

Ready to Run. 
It's simple! Run playbook now.

```sh
> ansible-playbook -i inventory/ mongodb.yml
```

#### 5. (Optional) Backup - Run playbook!

backup roles use the same variable file with `mongodb.yml`, fill out backup configuration part!
`spaceone/ansible/inventory/group_vars/mongodb.yml`
```yaml
# Backup Configuration
backup_or_not: # True | False

region_of_S3: "" # <region of backup bucket>
bucket_name: "" # <backup bucket name>
...
```

Run playbook!
```sh
> ansible-playbook -i inventory/ backup.yml
```