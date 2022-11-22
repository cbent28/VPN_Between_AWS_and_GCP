Building a VPN Between Google Cloud and AWS with Terraform
==========================================================


Google Cloud self-paced labs logo
![image](https://user-images.githubusercontent.com/104800728/203363728-fe49d047-c0a8-4a80-bd8e-204af0c2c1b1.png)

Overview
-----------------
This lab will show you how to use Terraform by HashiCorp to create secure, private, site-to-site connections between Google Cloud and Amazon Web Services (AWS) using virtual private networks (VPNs). This is a multi-cloud deployment.

In this lab, you will deploy virtual machine (VM) instances into custom virtual private cloud (VPC) networks in Google Cloud and AWS. You then deploy supporting infrastructure to construct a VPN connection with two Internet Protocol security (IPsec) tunnels between the Google Cloud and AWS VPC networks. The environment and tunnel deployment usually completes within four minutes. This lab is based off of the Automated Network Deployment tutorial.

Deployment architecture
-----------------------

In this lab, you build the following deployment environment:

![image](https://user-images.githubusercontent.com/104800728/203369852-959afc9d-fb13-4eab-994f-8692cbe37af6.png)

Objectives
---------

In this lab, you will:

- Build custom VPC networks with user-specified CIDR blocks in Google Cloud and AWS
- Deploy a VM instance in each VPC network
- Create VPN gateways in each VPC network and related resources for two IPsec tunnels
While Google Cloud uses routes to support equal-cost multi-path (ECMP) routing, AWS supports VPN gateways with two tunnels, active and standby, for redundancy and availability.

Routing
----------

The lab configuration uses Cloud Router to demonstrate dynamic routing. Cloud Router exchanges your VPC network route updates with your environment in AWS using Border Gateway Protocol (BGP). Dynamic routing by Cloud Router requires a separate Cloud Router for each IPsec tunnel. Alternatively, you can configure a setup with static routes. Both configurations are covered in the Cloud VPN Interop Guide.

Setup and requirements
=================================

Before you click the Start Lab button
------------------------------------------

Read these instructions. Labs are timed and you cannot pause them. The timer, which starts when you click Start Lab, shows how long Google Cloud resources will be made available to you.

This hands-on lab lets you do the lab activities yourself in a real cloud environment, not in a simulation or demo environment. It does so by giving you new, temporary credentials that you use to sign in and access Google Cloud for the duration of the lab.

To complete this lab, you need:

- Access to a standard internet browser (Chrome browser recommended).
Note: Use an Incognito or private browser window to run this lab. This prevents any conflicts between your personal account and the Student account, which may cause extra charges incurred to your personal account.
- Time to complete the lab---remember, once you start, you cannot pause a lab.
Note: If you already have your own personal Google Cloud account or project, do not use it for this lab to avoid extra charges to your account.

How to start your lab and sign in to the Google Cloud Console
------------------------------------------------------

1. Click the Start Lab button. If you need to pay for the lab, a pop-up opens for you to select your payment method. On the left is the Lab Details panel with the following:

The Open Google Console button
Time remaining
The temporary credentials that you must use for this lab
Other information, if needed, to step through this lab

2. Click Open Google Console. The lab spins up resources, and then opens another tab that shows the Sign in page.

Tip: Arrange the tabs in separate windows, side-by-side.

Note: If you see the Choose an account dialog, click Use Another Account.

3. If necessary, copy the Username from the Lab Details panel and paste it into the Sign in dialog. Click Next.

4. Copy the Password from the Lab Details panel and paste it into the Welcome dialog. Click Next.

Important: You must use the credentials from the left panel. Do not use your Google Cloud Skills Boost credentials.
Note: Using your own Google Cloud account for this lab may incur extra charges.

5. Click through the subsequent pages:

Accept the terms and conditions.
Do not add recovery options or two-factor authentication (because this is a temporary account).
Do not sign up for free trials.

After a few moments, the Cloud Console opens in this tab.

Note: You can view the menu with a list of Google Cloud Products and Services by clicking the Navigation menu at the top-left. Navigation menu icon

![image](https://user-images.githubusercontent.com/104800728/203410460-7d5b026a-3cae-4be5-947b-1598f5af78ea.png)

Activate Cloud Shell
------------------------

Cloud Shell is a virtual machine that is loaded with development tools. It offers a persistent 5GB home directory and runs on the Google Cloud. Cloud Shell provides command-line access to your Google Cloud resources.

1. Click Activate Cloud Shell Activate Cloud Shell icon at the top of the Google Cloud console.
When you are connected, you are already authenticated, and the project is set to your PROJECT_ID. The output contains a line that declares the PROJECT_ID for this session:

```
Your Cloud Platform project in this session is set to YOUR_PROJECT_ID
```

gcloud is the command-line tool for Google Cloud. It comes pre-installed on Cloud Shell and supports tab-completion.

2. (Optional) You can list the active account name with this command:

```
gcloud auth list
```

3. Click Authorize.

4. Your output should now look like this:

Output:

```
ACTIVE: *
ACCOUNT: student-01-xxxxxxxxxxxx@qwiklabs.net
To set the active account, run:
    $ gcloud config set account `ACCOUNT`
 ```
 
5. (Optional) You can list the project ID with this command:

```
gcloud config list project
```

Output:

[core]
project = <project_ID>

Example output:

[core]
project = qwiklabs-gcp-44776a13dea667a6

Note: For full documentation of gcloud, in Google Cloud, refer to the gcloud CLI overview guide.

Task 1. Preparing your Google Cloud working environment
======================================

In this section, you will clone the tutorial code and verify your Google Cloud region and zone.

Clone the tutorial code
------------------------------------------------
1. In the Google Cloud Console, open a new Cloud Shell window and copy the tutorial code:

```
gsutil cp gs://spls/gsp854/autonetdeploy-multicloudvpn2.tar .
tar -xvf autonetdeploy-multicloudvpn2.tar
```

2. Navigate to the tutorial directory:

```
cd autonetdeploy-multicloudvpn
```

Verify the Google Cloud region and zone
----------------------------------------

Certain cloud resources in this lab, including Compute Engine instances, VPN gateways, and Cloud Router, require you to explicitly declare the intended placement region or zone, or both. For more details, refer to Regions and Zones for Google Cloud.

This lab requires only a single region for each provider. To optimize the connection between the two clouds, choose regions near each other. The following table lists the values set in the tutorial files terraform/gcp_variables.tf and terraform/aws_variables.tf.

<img width="641" alt="Screen Shot 2022-11-22 at 3 10 53 PM" src="https://user-images.githubusercontent.com/104800728/203411796-a0f7ea6d-8784-47d5-80e9-ab6da7f2aed3.png">

Task 2. Preparing for AWS use
==============================

In this section, you will verify your AWS region. For details about AWS regions, refer to Regions and Availability Zones for AWS.

1. Sign in to the AWS Management Console (Click the Open AWS Console button on the left, and log in with the provided username and password).

2. Navigate to the EC2 Dashboard (Services > Compute > EC2). Select the Northern Virginia region (us-east-1) using the pulldown menu in the top right. In the EC2 Dashboard and the VPC Dashboard, you can review the resources deployed later in the lab.

Task 3. Creating access credentials
===================================

In this section, you will create access credentials for Google Cloud and AWS and point your templates at these access credentials.

Download Compute Engine default service account credentials
---------------------------------------------------

In Cloud Shell, which is a Linux environment, gcloud manages credentials files under the ~/.config/gcloud directory. To set up your Compute Engine default service account credentials, follow these steps:

1. In the Google Cloud Console, in the Navigation Menu Navigation menu icon , click IAM & Admin > Service Accounts.

2. Click the Compute Engine default service account , click on three vertical dots under Actions and select Manage keys, and click ADD KEY > Create new key.

3. Verify JSON is selected as the key type and click Create, which downloads your credentials as a file named [PROJECT_ID]-[UNIQUE_ID].json. Click CLOSE.

Click Check my progress to verify the objective.

4. In your Cloud Shell terminal, verify you are still in the autonetdeploy-multicloudvpn folder.

5. To upload your downloaded JSON file from your local machine into the Cloud Shell environment, click More more and click Upload then choose your downloaded file and click Upload.

6. Navigate to the JSON file you downloaded and click Open to upload. The file is placed in the home (~) directory.

7. Use the ./gcp_set_credentials.sh script provided to create the ~/.config/gcloud/credentials_autonetdeploy.json file. This script also creates terraform/terraform.tfvars with a reference to the new credentials.

Note: Replace [PROJECT_ID]-[UNIQUE_ID] with the actual file name of your downloaded JSON key.

```
./gcp_set_credentials.sh ~/[PROJECT_ID]-[UNIQUE_ID].json
```

Output:

```
Created ~/.config/gcloud/credentials_autonetdeploy.json from ~/[PROJECT_ID]-[UNIQUE_ID].json.
Updated gcp_credentials_file_path in ~/autonetdeploy-startup/terraform/terraform.tfvars.
```

Create AWS access credentials
-------------------------------

In this section, you will set up your Qwiklabs-generated AWS access credentials to use with Terraform. Note that the method used here differs from that of a production or personal environment due to lab constraints. If you would like to see how this is done outside of a lab environment, you can check out the steps in the Download Compute Engine default service account credentials documentation.

1. Run the following commands to create your credentials directory and file:

```
export username=`whoami`
mkdir /home/$username/.aws/
touch /home/$username/.aws/credentials_autonetdeploy
```

2. Run the following command to edit the credentials file. This is where you will put your Qwiklabs generated AWS Access and Secret keys.

```
nano /home/$username/.aws/credentials_autonetdeploy
```

3. On the first line, paste the following:

```
[default]
```

4. On the next line, add the following code. Replace <Your AWS Access Key> with your AWS Access Key from the Qwiklabs connection details panel.

```
aws_access_key_id=<Your AWS Access Key>
```

5. On the next line, add the following code. Replace <Your AWS Secret Key> with your AWS Secret Key from the Qwiklabs connection details panel.

```
aws_secret_access_key=<Your AWS Secret Key>
```

6. To save the file, use CTRL+X to exit, then type Y to save the changes, and hit Enter.

7. Run the following to inspect your security credentials file:

```
cat /home/$username/.aws/credentials_autonetdeploy
```

Your file should resemble the following:

```
[default]
aws_access_key_id=AKIA3INBXVI72ZO2Z4F4
aws_secret_access_key=bvQ+aMscVps34Q5ZZnazUGB2+kneKFr73P33iZIo
```

8. Once your file is correctly formatted, use the following command to set a Terraform environment variable to the correct AWS credentials file path:

```
export TF_VAR_aws_credentials_file_path=/home/$username/.aws/credentials_autonetdeploy
```

Note: A different way to do this would be to update the terraform/terraform.tfvars file to reference the AWS credentials file path.

Task 4. Setting your project
============================
In this section, you point your deployment templates at your project. Google Cloud offers several ways to designate the Google Cloud project to be used by the automation tools. For simplicity, instead of pulling the Project ID from the environment, the Google Cloud project is explicitly identified by a string variable in the template files.

1. Set your Google Cloud Project ID by using the following commands:

```
export PROJECT_ID=$(gcloud config get-value project)
gcloud config set project $PROJECT_ID
```

2. Use the provided script to update the project value in your configuration files for Terraform:

```
./gcp_set_project.sh
```

3. Review the updated file to verify that your project-id value has been inserted into terraform/terraform.tfvars. You can either use the cat command or the Cloud Shell Editor to verify the file.

4. Run the one-time terraform init command to install the Terraform providers for this deployment:

```
cd terraform
terraform init
```

5. Run the Terraform plan command to verify your credentials:

```
terraform plan
```

Output:

```
Refreshing Terraform state in-memory prior to plan...
...
 +google_compute_instance.gcp-vm
...
Plan: 34 to add, 0 to change, 0 to destroy.
```

Task 5. Using SSH keys for connecting to VM instances
=================================================

In Google Cloud, the Cloud Console and gcloud tool work behind the scenes to manage SSH keys for you.

- Use the ssh command to communicate with your Compute Engine instances without generating or uploading any key files.

However, for your multi-cloud exercises, you do need a public/private key pair to connect with VM instances in Amazon Elastic Compute Cloud (EC2).

Generate a key pair
---------------------

1. In Cloud Shell, use ssh-keygen to generate a new key pair:

```
ssh-keygen -t rsa -f ~/.ssh/vm-ssh-key -C $username
```

When asked for a passphrase, press Enter twice to leave it blank.

2. Restrict access to your private key. This is a best practice.

```
chmod 400 ~/.ssh/vm-ssh-key
```

Import the public key to Google Cloud
---------------------------------------

In this section, you will import and register your key.

1. In Cloud Shell, register your public key with Google Cloud:

```
gcloud compute config-ssh --ssh-key-file=~/.ssh/vm-ssh-key
```

Note: You can ignore the warning No host aliases were added... because the command also attempts to update Compute Engine VM instances, but no instances have been created yet.

2. In the Cloud Console, navigate to the Compute Engine > Metadata page.

3. Click SSH Keys. Verify your SSH Key exists.

4. Under the Key Section, copy the SSH Key value. You will use this in the next section.

![image](https://user-images.githubusercontent.com/104800728/203413725-63e060c1-21df-476f-b9ba-ffb36330b16d.png)

Import the public key to AWS
-------------------------

You can reuse the public key file generated with Google Cloud.

1. In the AWS Management Console, navigate to Services > Compute > EC2.
Note: Verify that you are in the US-East (N. Virginia) us-east-1 region.
2. In EC2 Dashboard, under the Network & Security group on the left, click Key Pairs.

3. Click Actions > Import Key Pair.

4. For the name, enter: vm-ssh-key.

5. Paste the contents of your Google Cloud public key (Compute Engine > Metadata > SSH Keys) into the Public key contents box.

6. Verify that the contents are of the expected form: ssh-rsa [KEY_DATA] [USERNAME].

7. Click Import Key Pair.

You can now see the AWS entry for the vm-ssh-key value. You can reference this key in configuration settings, which enables you to use the ssh command for access.

When you use the ssh command to connect to AWS instances, AWS behaves differently from Google Cloud. For AWS, you must provide a generic username supported by the AMI provider. In this tutorial, the AMI provider expects ubuntu as the user.

You now have an environment in which you can easily deploy resources to the cloud using automated tools. Use this environment to complete the rest of the lab.

Task 6. Examining Terraform configuration files
=================================================

In Terraform, a deployment configuration is defined by a directory of files. Although these files can be JSON files, it's better to use the Terraform configuration file (.tf file) syntax, which is easier to read and maintain. This lab provides a set of files that illustrate one way of cleanly organizing your resources. This set is a functional deployment and requires no edits to run.

<img width="638" alt="Screen Shot 2022-11-22 at 3 24 35 PM" src="https://user-images.githubusercontent.com/104800728/203414189-69281e60-a36e-49b9-bf8e-fc87b46d3337.png">

<img width="641" alt="Screen Shot 2022-11-22 at 3 24 48 PM" src="https://user-images.githubusercontent.com/104800728/203414162-0f77fc49-90aa-42b9-ab4b-815bacb970eb.png">

Task 7. Deploying VPC networks, VM instances, VPN gateways, and IPsec tunnels
=============================================

Constructing connections between multiple clouds is complex. You can deploy many resources in parallel in both environments, but when you are building IPsec tunnels, you need to order interdependencies carefully. For this reason, establishing a stable deployment configuration in code is a helpful way to scale your deployment knowledge. The following figure summarizes the steps required to create this deployment configuration across multiple providers.

![image](https://user-images.githubusercontent.com/104800728/203414314-6f684d21-df65-48f9-8bba-bd985df3c02d.png)

Task 8. Deploy with Terraform
===============================

Terraform uses the terraform.tfstate file to capture the resource state. To view the current resource state in a readable form, you can run terraform show.

1. In Cloud Shell, navigate to the terraform directory:

```
cd ~/autonetdeploy-multicloudvpn/terraform
```

2. Use the Terraform validate command to validate the syntax of your configuration files. This validation check is simpler than those performed as part of the plan and apply commands in subsequent steps. The validate command does not authenticate with any providers.

```
terraform validate
```

If you don't see an error message, you have completed an initial validation of your file syntax and basic semantics. If you do see an error message, the validation failed.

3. Use the Terraform plan command to review the deployment without instantiating resources in the cloud. The plan command requires successful authentication with all providers specified in the configuration.

```
terraform plan
```

The plan command returns an output listing of resources to be added, removed, or updated. The last line of the plan output shows a count of resources to be added, changed, or destroyed:

```
Refreshing Terraform state in-memory prior to plan...
...
Plan: 34 to add, 0 to change, 0 to destroy.
```

4. Use the Terraform apply command to create a deployment:

```
terraform apply
```

The apply command creates a deployment with backing resources in the cloud. In around four minutes, apply creates 30+ resources for you, including GCP and AWS VPC networks, VM instances, VPN gateways, and IPsec tunnels. The output of the apply command includes details of the resources deployed and the output variables defined by the configuration.

5. Type yes then enter to approve.

Click Check my progress to verify the objective.
Deploy with Terraform

6. Your deployments can emit output variables to aid your workflow. In this tutorial, the assigned internal and external IP addresses of VM instances have been identified as output variables by the gcp_outputs.tf and aws_outputs.tf files. These addresses are printed automatically when the apply step completes. If, later in your workflow, you want to redisplay the output variable values, use the output command:

```
terraform output
```

The output variables defined by this configuration include the internal and external IP addresses for your VM instances. To use the ssh command for network validation, you need the external IP addresses to connect to the VM instances.

```
aws_instance_external_ip = [AWS_EXTERNAL_IP]
aws_instance_internal_ip = 172.16.0.100
gcp_instance_external_ip = [GCP_EXTERNAL_IP]
gcp_instance_internal_ip = 10.240.0.100
```

7. Use the Terraform show command to inspect the deployed resources and verify the current state:

```
terraform show
```

8. To review your instances, use gcloud compute instances list, or use the Cloud Console, on the VM instances panel:

```
gcloud compute instances list
```

Output:

```
NAME                ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP    EXTERNAL_IP    STATUS
gcp-vm-us-central1  us-central1-a  n1-highmem-4               10.240.0.100   [EXTERNAL IP]  RUNNING
```

9. Verify that your Google Cloud VM instance is functioning by using the ssh command to connect to it:

```
ssh -i ~/.ssh/vm-ssh-key [GCP_EXTERNAL_IP]
```

You will get a message asking to confirm the authenticity of the host. Type yes.

10. Run the ping and curl commands in your ssh session:

```
ping -c 5 google.com
curl ifconfig.co/ip
```

11. Run simple network performance checks from the Google Cloud VM instance. Use pre-installed scripts to run a test on each network interface, both external and internal.

- Over external IPs:

```
/tmp/run_iperf_to_ext.sh
```

This script runs a 30-second performance test that produces summary data about network performance.

Resulting output:

```
...
[SUM]   ... sender
[SUM]   ... receiver
...

```

- Over VPN (internal IPs):

```
/tmp/run_iperf_to_int.sh
```

This script runs a 30-second performance test that produces summary data about network performance.

Resulting output:

```
...
[SUM]   ... sender
[SUM]   ... receiver
...
```

Note: Once the output has been displayed once, exit iperf.

12. When you complete the checks from the Google Cloud VM instance, enter the following command:

```
exit
```

13. To verify that your AWS VM instance is functioning, use the ssh command to connect to it:

```
ssh -i ~/.ssh/vm-ssh-key ubuntu@[AWS_EXTERNAL_IP]
```
You will get a message asking to confirm the authenticity of the host. Type yes.

14. Run the ping and curl commands in your ssh session:

```
ping -c 5 google.com
curl ifconfig.co/ip
```

15. Run simple network performance checks from the AWS VM instance. Use pre-installed scripts to run a test on each network interface, both external and internal.

- Over external IPs:

```
/tmp/run_iperf_to_ext.sh
```

This script runs a 30-second performance test that produces summary data about network performance.

Resulting output:

```
...
[SUM]   ... sender
[SUM]   ... receiver
...
```

- Over VPN (internal IPs):

```
/tmp/run_iperf_to_int.sh
```

This script runs a 30-second performance test that produces summary data about network performance.

Output:

```
...
[SUM]   ... sender
[SUM]   ... receiver
...
```

Note: Once the output has been displayed once, exit iperf.

When you complete the checks from the AWS VM instance, enter the following command:

```
exit
```

You have successfully deployed a secure, private, site-to-site connection between Google Cloud and AWS using VPN! You can optionally destroy your infrastructure using the terraform destroy command, explore the resources you created, or end your lab here.

Congratulations!
====================

In this lab, you used Terraform to build custom VPC networks with user-specified CIDR blocks in Google Cloud and AWS, deploy a VM instance in each network, and create VPN gateways in each VPC network and related resources for two IPsec tunnels.

Next steps / Learn more
------------------------

Be sure to check out the following to receive more hands-on practice with Terraform:

- Hashicorp Learn
- Terraform Enterprise on the Google Cloud Marketplace
- Terraform Community
- Terraform Google Examples

Google Cloud training and certification
-------------------------------------------

...helps you make the most of Google Cloud technologies. Our classes include technical skills and best practices to help you get up to speed quickly and continue your learning journey. We offer fundamental to advanced level training, with on-demand, live, and virtual options to suit your busy schedule. Certifications help you validate and prove your skill and expertise in Google Cloud technologies.

Manual Last August 20, 2022

Lab Last May 20, 2022

Copyright 2022 Google LLC All rights reserved. Google and the Google logo are trademarks of Google LLC. All other company and product names may be trademarks of the respective companies with which they are associated.
