# Cuckoo SandBox on AWS
By Oran Kabarity @ Check Point Software Technologies

## Overview

The project is an extension to Cuckoo Sandbox open source project; it adds support to AWS cloud functionalities and enables running emulations on auto-scaling infrastructure.

This blog post explains in detail the theory and functionality of Cuckoo Sandbox over AWS cloud.  
https://research.checkpoint.com/cuckoo-system-on-aws/

## Installation instructions - Nest Setup


•	Launch a Ubuntu Linux machine (possible via AWS marketplace)

Update the package indexes and packages
```
sudo apt update && sudo apt upgrade
```

•	Clone the repository
```
git clone https://github.com/CheckPointSW/Cuckoo-AWS
```
•	Install pre-requisite packages
```
sudo apt install virtualenv vim python
```
•	Setup and activate virtual environment 
```
sudo virtualenv venv
. venv/bin/activate
```
Install some dependencies:
```
sudo apt install python-dev libffi-dev libssl-dev build-essential libjpeg8-dev zlib1g-dev
```
•	Install boto3 library
```
pip install boto3
```

•	Obtain the matching monitoring binaries from the community repository (execute from the Cuckoo-AWS directory)
```
python stuff/monitor.py
```

•	Install cuckoo as DEV mode
```
python setup.py sdist develop
```

•	Run cuckoo with debug output
```
cuckoo –d
```

•	The first run should build the configuration files and save them in some location. The location is shown in the output of the run (should contain “.cuckoo” library). It is strongly advised to remember that location for the following steps and for future usages

•	Edit "~/.cuckoo/conf/cuckoo.conf" and add the Private IP address, the AWS VPC private IP address assigned to this instance.
```
machinery = aws
[resultserver] ip = <the private IP of this machine>
```

•	Edit "~/.cuckoo/conf/aws.conf" according to the instructions in the file

•	Run 
```
cuckoo 
```
 

## Image and snapshot setup

•	Cuckoo are recommending to run guest VMs on 64-bit Windows 7. You can import your regular VM to AMI using this manual.
  https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html
  
  The above manual is a bit dated, for Windows 10, use Group Policy Editor to disable windows Defender and Windows Updates, alternately we can write an IPS signature to block files from being downloaded, but still allow updates from Microsoft to further evade detection. 

•	Launch a new Windows 7 instance to setup the guest 

•	Install Python and Pillow library (https://cuckoo.sh/docs/installation/guest/requirements.html)

For Windows, pre-packaged with pip, !be sure to check add Python to PATH!:
```
https://www.python.org/ftp/python/2.7.16/python-2.7.16.amd64.msi
```
Install Pillow
```
pip install Pillow
```

•	Disable Windows Firewall and the Automatic Updates

From a command prompt execute:
```
GPEDIT.msc
```
Locate the following setting and set it to disbaled:
```
    Computer Configuration
        Policies
            Administrative Templates: Policy …
                Network
                    Network COnnections
                        Windows Firewall
                            Domain Profile
                               Double-click the “Windows Firewall: Protect all network connections” object and change the setting to "Disabled"
```
Next, locate the following and set it to enabled as follows:
```
     Computer Configuration
          Administrative Templates
               Windows Components
                    Windows Updates
                         Click on Edit policy setting to open the Configure Automatic Updates dialog.
                              On the Configure Automatic Updates dialog, select Enabled in the left pane, in the Options section click on the Configure Automatic Updating Combo Box and in the dropdown list select "Notify for download and notify for install."
```  
•	Install Cuckoo agent (https://cuckoo.sh/docs/installation/guest/agent.html)
From the Cuckoo directoy located in "~/.cuckoo/agent/" copy the agent.py filw to the windows macine and rename it to <randomstring>.pyw (pyw executes without displaing the console durring execution)
 
 Copy this file to the Windows "Stat-up" folder. (On Windows 10 this is located at :\Users\Username\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup)
 

•	For malware network analysis, each guest should have the Nest as their default route 

•	Save this instace as a new image(AMI). This action will also create a new snapshot(snapshot-id can be found under the AMI details)

To export this vm to AWS:
1. In virtualbox, right click on the VM, and choose "Export this VM to OVI"
```
Format: Open Virtuialization Format 2.0
Strip all MAC addresses
Check, "Write Manifest File"
Click "Next" and "Export"
```
2. Upload the Image to AWS
```
Create and IAM user "cuckoo"
Add S3 Admin Permissions for this user (can be done through AWS Management Console)
```
Wait on the screen that has the API keys and run from CMD prompt:

```
aws configure
```
Create an S3 Bucket:
```
aws s3 mb s3://cockoo --region us-east-1
```
Upload the OVA image created above:
```
aws s3 cp Windows10.ova s3://cuckoo --region us-east-1
```
Next we will create some cinfogruation files on our computer to send to AWS via the AWS CLI. This next one is to add the ability to create a vimimport to the user "cuckoo" that we just created to upload files to S3, the first step ios to create the policy:
```
notepad trust-policy.json
```
Paste in the following:
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```
Next create a role and add the security policy we just created to the role:
```
aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json
```

Next, create a role-policy.json file and specify our bucket we are using for import:
```
notepad role-policy.json
```
paste:
```
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
            "s3:ListBucket",
            "s3:GetBucketLocation"
         ],
         "Resource": [
            "arn:aws:s3:::cuckoo"
         ]
      },
      {
         "Effect": "Allow",
         "Action": [
            "s3:GetObject"
         ],
         "Resource": [
            "arn:aws:s3:::cuckoo/*"
         ]
      },
      {
         "Effect": "Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource": "*"
      }
   ]
}
```
Next add this policy to the vmimport role:
```
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json
```
Next create a small config that tells vmimport what to import and from where:
```
notepad containers.json
```
Paste:
```
[{ "Description": "Windows 10 Base", "Format": "ova", "UserBucket": { "S3Bucket": "cuckoo", "S3Key": "Windows10.ova" } }]
```
Finally, create our AMI on EC2 from this image:
```
aws ec2 import-image --description "Windows 10" --disk-containers file://containers.json --region ue-east-1
```
This is going to take a while, you can check on the status usignt he following command:
```
aws ec2 describe-import-image-tasks --region us-east-1
```

## Problems and solutions
•	In case that the installation fails or if the following exception appears ” 'module' object has no attribute 'get_installed_distributions' ”, try downgrading pip:
```
pip install --force-reinstall pip==9.0.3
```

•	In case of various issues during the build of the configuration files, try re-generating the configuration. 
Delete the whole “.cuckoo” folder and run the following:
```
cuckoo –d 
```

## Changes from official Cuckoo repository

In order to make it compatible with AWS we made the following changes.

•	Added a new machinery (at cuckoo/machinery/aws.py)

•	Added a new config template (at cuckoo/private/cwd/conf/aws.conf)

•	Modified the config object (at cuckoo/common/config.py) in a way that will support the new configuration file.
