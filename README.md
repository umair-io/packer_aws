
# Creating and provisioning AMIs using Packer

  

## Intro

### What is Packer?
HashiCorp Packer is easy to use and automates the creation of any type of machine image and in this instance AMI (Amazon Machine Image) and thus Packer can be used to "bake"/provision AMIs.

### What is an AMI?

An Amazon Machine Image (AMI) provides the information required to launch an instance. You must specify an AMI when you launch an instance. You can launch multiple instances from a single AMI when you need multiple instances with the same configuration. You can use different AMIs to launch instances when you need instances with different configurations. In simple words, AMI is a starting point for a cloud server in AWS.
  

### What are we going to do today?

We are going to be building our first AMI using packer and then provisioning a simple web server on it using packer in to the AMI. Then we are going to start an AWS instance using that AMI we created.

 
 Lets get started!
  
## The AWS Stuff

### Setup AWS account

#### Create AWS account (free-tier)
* Go to https://aws.amazon.com/free/ and follow instructions to sign up to AWS. (Debit/Credit card required)
  
#### Create AWS access key for CLI access
* Go to AWS IAM Console and Create a User. [Direct link](`https://console.aws.amazon.com/iam/home?region=eu-west-2#/users`)
* Type username and select 'Programmatic access'
* Give user 'AdministratorAccess' from the existing policies.
* Keep clicking next and on the last page click 'Create user'
* Download .csv file. We will need it to allow terraform to access our AWS account.

### Local AWS config
#### Install AWS cli (Windows)
- Follow [this guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html)

#### Install AWS cli (Mac)
- Run following commands in terminal:
```shell
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

#### Set AWS credentials
* Run the following command and fill in the following details

```shell
PS C:\Users\Umair> aws configure
AWS Access Key ID [None]: [YOUR 'Access key ID' IN DOWNLOADED CSV]
AWS Secret Access Key [None]: [YOUR 'Secret access key' IN DOWNLOADED CSV]
Default region name [None]: eu-west-2
Default output format [None]:
```
This will now allow packer to automatically pick you AWS creds from ~/.aws/credentials

### Select a base AMI (using AWS console)
We need an AMI that packer can use a source to create AMIs. So lets head over to AWS console to pick a candidate AMI. 

Here's a [direct link]( https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=public-images;sort=name)

Amazon Linux 2 is a good AMI to use as a source so that's what we will go for and it's AMI ID is `ami-01a6e31ac994bbc09`

  
  ## Packer
  
### Install Packer
* For Windows, follow [this guide](https://learn.hashicorp.com/packer/getting-started/install)
* For Mac, run the following commands:
```
# Install brew
`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`
# Install packer (using brew)
brew install packer
```

  
### Build an AMI using Packer

We'll start by creating the entire template, then we'll go over each section briefly. Create a file **packer-basic-example.json** and fill it with the following contents:

```json
{
	"builders": [
	  {
		"type": "amazon-ebs",
		"region": "eu-west-2",
		"source_ami" : "ami-01a6e31ac994bbc09",
		"instance_type": "t2.micro",
		"ssh_username": "ec2-user",
		"ami_name": "packer-basic-example {{timestamp}}"
	 }
	]
}
```

Lets analyse above template:

*  `type`: We want an Amazon EBS (Elastic Block Store) based image.
*  `region`: Which AWS region we want to builder (a temporary server used to create the AMI) to start in. We have gone for eu-west-2 which London region.
*  `source_ami`: That's the Amazon Linux 2 ami we choose up there to use as a base. You can also optionaly used `source_ami_filters` to search for AMIs using names and Owners rather than id.
*  `ssh_username`: The username to connect to SSH with.
*  `ami_name`: The name of the output AMI.

Let's run the above template now
```
packer packer-basic-example.json
```

Analyse the output and then head over to see out AMI. Here's a [direct link](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=private-images;search=packer-basic-example;sort=name) to your new ami.
  

### Lets add some provisioning to it (using Bash)
Creating another AMI which is nearly identical to the source which just a new name is boring. So we are going to bake in a web server in to this AMI using packer to make it a bit more special.  Create a file **packer-provision-example.json** and fill it with the following contents:

```json
{
	"builders": [
	  {
		"type": "amazon-ebs",
		"region": "eu-west-2",
		"source_ami" : "ami-01a6e31ac994bbc09",
		"instance_type": "t2.micro",
		"ssh_username": "ec2-user",
		"ami_name": "packer-provision-example {{timestamp}}"
	 }
	],
	"provisioners": [
	  {
		"type": "shell",
		"inline": [
		  "sudo yum -y install httpd"
		 ]
	  }
	] 
}
```
Let's run the above template now
```
packer packer-provision-example.json
```
Analyse the output and then head over to see out AMI. Here's a [direct link](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=private-images;search=packer-provision-example;sort=name) to your new ami.

### Add some more provisioning to it (using Ansible)
This bit is only for Unix (Mac/Linux) users. Pre-req for this is have ansible installed. I'll let you figure it out by checking out the [this link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-macos)
	
Let's add some more web pages to this our web server. Create a file **packer-provision-ansible-example.json** and fill it with the following contents:
```json
{
	"builders": [
	  {
		"type": "amazon-ebs",
		"region": "eu-west-2",
		"source_ami" : "ami-01a6e31ac994bbc09",
		"instance_type": "t2.micro",
		"ssh_username": "ec2-user",
		"ami_name": "packer-provision-ansible-example {{timestamp}}"
	 }
	],
	"provisioners": [
	  {
		"type": "shell",
		"inline": [
		  "sudo yum -y install httpd"
		 ]
	  }
	] 
}
```
Let's run the above template now
```
packer packer-provision-ansible-example.json
```
Analyse the output and then head over to see out AMI. Here's a [direct link](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=private-images;search=packer-provision-ansible-example;sort=name) to your new ami.
