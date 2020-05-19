
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
* Go to AWS IAM Console and Create a User. [Direct link](https://console.aws.amazon.com/iam/home?region=eu-west-2#/users)
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
umair@Umairs-MBP aws configure
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

Let's check we don't have any syntax issues with our template by running following. It should return `Template validated successfully.`
```
packer validate packer-basic-example.json
```

Time to build our AMI using following the command:
```
packer build packer-basic-example.json
```

While the build is running, monitor the terminal to see what is packer doing. Also check out [AWS EC2 Instaces page](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Instances:sort=instanceState) to see how packer creates a temporary builder ec2 instance/server to create new AMI and once done, terminates it. 

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
            "sudo yum install -y httpd",
            "sudo chkconfig httpd on",
            "sudo service httpd start",
            "sudo chown -R ec2-user:ec2-user /var/www/",
            "echo '<h1>Welcome to Packer</h1>' > /var/www/html/index.html"
		 ]
	  }
	] 
}
```

Let's validate the template then build it using packer:
```
packer validate packer-provision-example.json
packer build packer-provision-example.json
```

Analyse the output and then head over to see out AMI. Here's a [direct link](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=private-images;search=packer-provision-example;sort=name) to your new ami.


### Using our shiny new AMI
* Head over to AWS EC2 [homepage](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Home:).
* Click `Launch Instance`
* On `Choose AMI` step, click `My AMIs`. Now you should see the `packer-provsion-example` AMI that we made just now. Select it.
* Select `t2.micro` as your instance type. It's free!
* Now `Next` all the way to `6. Configure Security Group`
* Create a seurity group to allow http and ssh traffic.
  * Select `Create a new security group`
  * ` Security group name:` web
  * There should already be ssh rule there so let's keep it there.
  * Click `Add Rule` and select `HTTP` as the type. 
  * Keep everything as default and click `Review and Launch`
* On the Review page, click `Launch`.
* When prompted to Select a key pair, select `Create a new key pair` and give a name before downloading it. You can use to ssh in to the ec2 instance.
* Finally click `Launch Instance`.
* Wait for the instance to start up up and go green.
* Now find `Public DNS (IPv4)` or `Public DNS (IPv4)` for your new ec2 instance and stick it in your web browser address and check out our web server.


### Add some more provisioning to it (using Ansible)
This bit is only for Unix (Mac/Linux) users. Pre-req for this is have ansible installed. I'll let you figure it out by checking out [this link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-macos)
	
Using Ansible (a software provisioning tool), let's add some more web pages to this our web server and now we are going to bake this stuff on top on the AMI we made in the last step. If your a lot of existing automation is done using ansible, this is a great way to re-use your stuff. Check out`source_ami_filter` tag for new stuff!

Create a file **update_website.yml**, fill it with following contents and may be replace `YOUR_NAME` with your name to make it a bit more personal:
```yaml
---

- name: Update the webserver
  gather_facts: false
  hosts: all
  tasks:
    - name: Updating the /var/www/html/index.html
      lineinfile:
        dest=/var/www/html/index.html
        line="Well done YOUR_NAME, you have now done provisioning using ansible"
        owner=ec2-user
```

Before we create our template, we are going to find out our (12-digit) AWS account ID but running the following command. We'll need it in the template below:
```
aws sts get-caller-identity --query Account --output text
```

Create a file **packer-provision-ansible-example.json** and fill it with the following contents:
```json
{
	"builders": [
	  {
		"type": "amazon-ebs",
		"region": "eu-west-2",
		"source_ami_filter": {
			"filters": {
				"virtualization-type": "hvm",
				"name": "packer-provision-example*",
				"root-device-type": "ebs"
				},
				"owners": ["[YOUR_AWS_ACCOUNT_ID]"],
				"most_recent": true
		},
		"instance_type": "t2.micro",
		"ssh_username": "ec2-user",
		"ami_name": "packer-provision-ansible-example {{timestamp}}"
	 }
	],
	"provisioners": [
	  {
		"type": "ansible",
		"playbook_file": "update_website.yml"
	  }
	] 
}
```
Let's validate the template then build it using packer:
```
packer validate packer-provision-ansible-example.json
packer build packer-provision-ansible-example.json
```
Analyse the output and then head over to see out AMI. Here's a [direct link](https://eu-west-2.console.aws.amazon.com/ec2/v2/home?region=eu-west-2#Images:visibility=private-images;search=packer-provision-ansible-example;sort=name) to your new ami.

Now let's build a new ec2 instance with our new AMI. Use [instructions above](#using-our-shiny-new-ami) if you need a reminder on how to. *Use the existing "web" security group if you created one earlier using the instructions above.*

You should have a good basic understanding of how packer works and how it can be used. I'd suggest you continue your packer journey and try building some more images with more stuff baked in them!