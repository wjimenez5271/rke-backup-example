# RKE Backup and Restore Example

Setting up recurring backups for Rancher server is an important part of making Rancher production-ready. Rancher relies on etcd for persisting its configuration and state. Although etcd is a distributed data store, it is not impervious to hardware failure. Consistent backups is your best defense against this type of failure, so this example will help you understand how to set this up in your own instance of Rancher. 

Below is an example of automation to produce recurring backups in Rancher as well as the automation to perform a recovery from backup. First we are going to create an RKE cluster using terraform and EC2, and then we'll configure the backup job. 

## Requirements
 - Terraform
 - AWS
 - RKE

 If you are on a Mac you can install Terraform and RKE with Homebrew by running: `brew install terraform` & `brew install rke`.

## Configuring Backup

### Setp 1 - Start two EC2 Servers
First create a file called `terraform.tfvars` that contains the following: 

```
ssh_keypair = ""
vpc_id = "
resource_prefix = "my-name-or-unqiue-id"
```

Replace the variables with values that correspond to your environment. These values will be used by Terraform to build EC2 instances in your AWS account. You also should have a set of AWS credentials defined per the (TF module's instructions)[https://www.terraform.io/docs/providers/aws/index.html#authentication]


Then run: 

```
terraform init
```

and subsequently

```
terraform up
```

You should now have two Ubuntu servers running in EC2. Take note of their IP addresses for the next step.

### Step 2 - Configure RKE cluster.yml
Now we are going to configure RKE to connect to these servers to build a Kubernetes cluster. This is also where we will configure the backup job. Edit the file called `cluster.yml` in the root of this repo to reflect the correct IP addresses of the two servers you just created. 

Next we want to configure the backup settings for the cluster. Our Terraform plan from above created an S3 bucket we can use for this called `demo-rke-backup-bucket`. Update the section under the key `etcd` in the `cluster.yml` to reflect the correct settings for your AWS account. You'll need to provide it with AWS credentials that the automated job can use to upload. We recommend you create a narrow-scoped IAM user profile for this purpose. 

### Step 3 - Deploying Kubernetes and starting Backup Job
Now we can apply our settings with RKE:

```
rke up --ssh-agent-auth
```

This will install Kubernetes on these nodes and configure automated backup. You may need to add the SSH key you are using in AWS to your agent by running `ssh-add /path/to/key`

## Configuring Restore

In the event of a etcd failure or corruption, we can use our snapshot to restore the Rancher cluster state. Note that this is a destructive step to the existing etcd running in your cluster, so only do this if you want to replace the running state with the data in the backup. 

To preform a restore, we'll re-use much of the information from the previous example to construct the restore command: 

```
rke etcd snapshot-restore --config cluster.yml --name snapshot-name \
--s3 --access-key S3_ACCESS_KEY --secret-key S3_SECRET_KEY \
--bucket-name demo-rke-backup-bucket --s3-endpoint s3.amazonaws.com
```

Be sure to replace "snapshot-name" with the correct name of an actual snapshot. Also replace "access-key" and "secret-key" with the same keys we used previously to write the snapshots. 
