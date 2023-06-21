![architecture](/images/archtecture2.png?raw=true)

# Wickr Enterprise High-Availability Cloudformation


## Overview

This template and associated instructions will build a Wickr Enterprise High-Availability deployment across three Availability Zones and is designed to be as automated as possible. The deployment will take ~15mins and includes the following resources:

- 1 x Amazon Virtual Private Cloud (Amazon VPC)
- 3 x Public and 3 x Private Subnets, with 3 x NAT Gateways spanning 3 Availability Zones in the given region
- An Amazon Simple Storage Service (Amazon S3) bucket, encrypted with server-side encryption with Amazon S3 managed keys (SSE-S3) and access logging enabled
- An Amazon Elastic Compute Cloud (Amazon EC2) instance running Amazon Linux 2 with necessary tools pre-installed, to be used as a jump-box for cluster administration via AWS Systems Manager Session Manager
- An Amazon Elastic Kubernetes Service (Amazon EKS) cluster running EKS version 1.23
- An Amazon Relational Database Service (Amazon RDS) running Amazon Aurora for MySQL, encrypted with an AWS Key Management Service (KMS) Customer Managed Key (CMK)
- Amazon Aurora for MySQL admin user credential dynamically provisioned and stored in AWS Secrets Manager, secured with an AWS KMS CMK

## Requirements

- An existing Amazon EC2 KeyPair uploaded or created in your AWS Account. Instructions can be found [here.](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
- [Administrative](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html#jf_administrator) user/role with credentials for the AWS CLI that you are willing and able to use both locally and within the jump-box
- A valid Wickr Enterprise HA license key (.yaml format)
- L200/300 Linux CLI knowledge and L100/200 AWS CLI knowledge
- Cloned this repository locally and entered the repo directory
- The AWS Systems Manager Session Manager plugin [installed](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) and [configured](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html#ssh-connections-enable) on your local device that you will use to SSH into the jump-box
- Access to a DNS provider and (optionally) Administrative access to AWS Certificate Manager within your account

## Instructions

1. Assume your administrative AWS CLI user/role, instructions can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html). Run the following command to deploy the Cloudformation template. 
- **Replace** `SSHKeyName` with the name of your existing SSH key
- **Replace** `sourceIP` with the IP range that your Wickr clients will be connecting from (typically everywhere/anywhere, i.e 0.0.0.0/0)
```bash
aws cloudformation deploy --stack-name wickr-ent-ha --template-file cloudformation.yaml --parameter-overrides SSHKeyName=key sourceIP=10.0.0.0/0 --capabilities CAPABILITY_NAMED_IAM
```
2. Once the stack has deployed, **copy** your Wickr Enterprise HA license key to the EC2 jump-box created for you, e.g. via SCP using Session Manager. You can get the ID by going to the **Outputs** [tab](https://eu-west-2.console.aws.amazon.com/cloudformation) of the wickr-ent-ha Cloudformation stack. **Ensure** you leave the trailing ':' after the jump-box Id if using the SCP command below.
```bash
scp -i YOUR_PRIVATE_KEY YOUR_LOCAL_LICENCE_KEY ec2-user@[JUMP-BOX-ID]:
```
(N.B. You will need to first make sure the permissions on your SSH private key are not too open, i.e. you may need to run `chmod 600 YOUR_PRIVATE_KEY`)

3. **SSH** using Session Manager into the EC2 jump-box
```bash
ssh -i YOUR_PRIVATE_KEY ec2-user@JUMP-BOX-ID
```

4. **Very Important!** 
Within the EC2 jump-box, **temporarily assume the same administrative AWS CLI user/role as in Step 1**. Instructions can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html). 
The simplest method is just to paste temporary credentials into the CLI as environment variables.
Otherwise, if you are using the EC2 instance profile created by this Cloudformation template to assume your Admin IAM role, you will need to include the EC2 instance profile in the trust policy of your Admin IAM role and then export the temporary credentials as environment variables (note, a CLI _named profile_ will not work in Step 5) via the following bash command
(replace `YOUR_ACCOUNT` and `YOUR_ROLENAME`):
```bash
OUT=$(aws sts assume-role --role-arn arn:aws:iam::<YOUR_ACCOUNT>:role/<YOUR_ROLENAME> --role-session-name jump-box);\
export AWS_ACCESS_KEY_ID=$(echo $OUT | jq -r '.Credentials''.AccessKeyId');\
export AWS_SECRET_ACCESS_KEY=$(echo $OUT | jq -r '.Credentials''.SecretAccessKey');\
export AWS_SESSION_TOKEN=$(echo $OUT | jq -r '.Credentials''.SessionToken');
```
Run `aws sts get-caller-identity` to confirm you are in the correct role - the same role used in Step 1.

5. Once you have completed the step above, run the following command:
```bash
configure-cluster.sh
```
6. Follow the on-screen prompts and allow the command to finish.
7. The Wickr cluster admin console will now deploy. You will need to create a console password during this process which you must store securely.
8. Once you have been told to access the admin console, open a **new terminal tab** and assume your Administrative role again, then run the following command,
which will set up a port forwarding session over the SSH tunnel from your local machine to the remote jump-box, where the admin console is running.
- **NOTE:** DO NOT close your current tab, it must remain open whilst the admin console is in use!

```bash
aws ssm start-session --target JUMP-BOX-ID --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["8800"], "localPortNumber":["8800"]}'
```
9. Click on [this link](http://localhost:8800) to access the admin console via the tunnel.

## Configure Wickr Enterprise HA

1. You will need to fill in the following configuration fields in the Wickr Admin console:

* **Hostname** - Enter the endpoint for the Wickr client to connect to. This will be the hostname that your Wickr clients will connect to, as well as the Network Administrative console address.
* **Enable Global Federation** - Leave unchecked unless you have been advised to enable this feature.
* **Certificate Type** - Select to use an [Amazon Certificate Manager](https://console.aws.amazon.com/acm) (ACM) certificate or a self-signed certificate. If you opt to use ACM, you need to create the certificate using ACM in the AWS Console and use the ARN as the value. 
* **Set a Pinned Certificate** - If using ACM, you will need to supply the certificate Amazon Root CA in .pem format. You can get this from [here](https://www.amazontrust.com/repository/AmazonRootCA1.pem). Create a file from this (including the BEGIN and END statements!) and name it AmazonRootCA.pem
* **Database** - You can find the Database Hostname in the Cloudformation outputs tab, and the admin username and password in [AWS Secrets Manager](https://console.aws.amazon.com/secretsmanager/). Leave the other database fields empty as they will assume the defaults. 
* **S3** - You can find the bucket name in the Cloudformation outputs tab, and the region will be the region you have deployed into.

2. When finished, select **‘Continue’** at the bottom of the window. The console will run some preflight checks to ensure your cluster will meet the minimum requirements. The ‘Amazon S3 Permissions’ check may not pass right away, as it can take some time to connect. **NOTE:** If this happens, wait a minute, navigate to ‘Application’ on the top left of the page and select ‘Checks passed with warnings’ on the dashboard. Then select **‘Re-run’** - the check should pass now. 
3. Return to the Wickr dashboard tab, which should now show as deploying. Wickr will now deploy to the cluster. This will take ~15 mins. 
4. (Recommended) The deployment will have deployed a Network Load Balancer in your AWS account for you as part of the build, and associated your ACM certificate with it if you provided one. You will need to point your DNS entry to that NLB A-record.
5. You can now navigate to `https://[wickr-hostname]` to access the Wickr Network Administrative console. The default credentials are `admin` and `Password123` and you will be asked to change them on first login.

## Post-Installation 

- The EC2 jump-box is only required when you need to reach the cluster config screen again, so this can be powered down if desired.

## Notes ##

1. If you need to re-deploy the cluster admin console, simply re-assume the EKSAdmin role with the following command, ensuring you **replace** EKSAdminArn with the Arn of that role. You can get this by going to the **Outputs** tab of the wickr-ent-ha Cloudformation stack.
```bash
RAW_JSON=$(aws sts assume-role --role-arn EKSAdminArn --role-session-name eksadmin-cli --output json) && KEYID=$(echo $RAW_JSON | jq .Credentials.AccessKeyId | tr -d '"') && AKEY=$(echo $RAW_JSON | jq .Credentials.SecretAccessKey | tr -d '"') && TOKEN=$(echo $RAW_JSON | jq .Credentials.SessionToken | tr -d '"') && export AWS_ACCESS_KEY_ID="$KEYID" && export AWS_SECRET_ACCESS_KEY="$AKEY" && export AWS_SESSION_TOKEN="$TOKEN" && aws sts get-caller-identity
```
2. You should see a confirmation that you have assumed the EKSAdminRole, then run the following command.
```bash
kubectl kots admin-console --namespace wickr
```

## Cleanup ##

1. Delete the Application Load Balancer that has been deployed by the Wickr installation process
2. Empty the S3 bucket
3. Delete the eksctl-wickr-ha-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa Cloudformation stack
3. Delete the wickr-ent-ha Cloudformation stack

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
