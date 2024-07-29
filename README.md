# WordPress

## Description

WordPress running in ECS on EC2 with Bitnami container image. For file storage, it uses EC2 with additional EBS instead of EFS due to the speed of EFS depends on the amount of data, thus it is not suitable for small number of files. Additional EBS is used to save cost on backup. With this architecture, backup is only needs to be done on EBS and Aurora instead of the whole EC2. ECS is connected to file storage EC2 using NFS protocol. 

On scaling, it uses application and cluster autoscaler with target tracking strategy managed by ECS. 

Database uses Aurora MySQL for its compute and storage separation to reduce possibility of data loss when the compute crashes.

ECS template comes with lambda-backed custom resources to scale the number of EC2 to 0 before deletion because ECS cluster can't be deleted when there's EC2 instance(s) registered to it.

## Limitations

- Current template, especially EBS, is not highly available.
- NAT Gateway is configured only in 1 AZ, so it will incur data transfer cost for different AZ traffic.
- For simplicity sake, the custom resource has AutoScalingFullAccess AWS managed policy. For production, it is recommended to use custom least-privilege policy.

## Cost

- NAT Gateway + data processing
- EIP (public ip)
- EC2
- Aurora MySQL
- Data Transfer
- ALB

## Instructions

1. Create network stack using wordpress-network.yml. Specify the CIDR block if preferred (default to 10.0.0.0/16).
    
    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-network --template-body file://wordpress-network.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

    To specify custom CIDR block:

    <code>aws cloudformation create-stack --stack-name wordpress-network --template-body file://wordpress-network.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span> --parameters ParameterKey=CidrBlock,ParameterValue=<span style="color: red">172.16.0.0/20</span></code>

2. Create ALB stack using wordpress-alb.yml. By default, this is using HTTP and the site can be visited using ALB's DNS name if no custom domain name is specified.

    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-alb --template-body file://wordpress-alb.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

3. Create ecs-securitygroup stack using wordpress-ecs-securitygroup.yml. This is to enable database to allow ingress before the container is launched.

    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-ecs-securitygroup --template-body file://wordpress-ecs-securitygroup.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

4. Create secret manager cluster stack using wordpress-secret.yml. We will use this secret manager to store Aurora master username and password.

   **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-secret --template-body file://wordpress-secret.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

5. Use AWS CLI to store the master username and password.

   <code>aws secretsmanager put-secret-value --secret-id wordpress-aurora --secret-string "{\`"username\`":\`"<span style="color: red">aurora_username</span>\`",\`"password\`":\`"<span style="color: red">aurora_password</span>\`"}" --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

6. Create aurora stack using wordpress-aurora.yml. If custom domain preferred, specify it under AuroraCustomDomainName and provide value for HostedZoneId parameter.

    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-aurora --template-body file://wordpress-aurora.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

7. Create nfs stack using wordpress-nfs.yml.

    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-nfs --template-body file://wordpress-nfs.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

8. Create wordpress-ecs.yml stack.

    **AWS CLI:**

    <code>aws cloudformation create-stack --stack-name wordpress-ecs --template-body file://wordpress-ecs.yml --region <span style="color: red">region</span> --profile <span style="color: red">profile</span></code>

9. Visit the site:
    - If no custom domain name specified, use http:// with ALB's DNS name which can be found under ALB stack output.
    - If custom domain name is specified, use either http:// or https:// with custom domain name.

## Troubleshooting
If visiting the site doesn't work, check the following:
- Ensure that the ecs task is running and didn't crash after a short while.
- Ensure that the ALB target group health check is healthy. By default, healthcheck is configured to use HTTPS on port 8443.