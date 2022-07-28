# Granting Dynamic Access by Referencing Security Groups
## Preparation
### This recipe requires some “prep work” which deploys resources that you’ll build the solution on. You will use the AWS CDK to deploy these resources 

### In the root of this Chapter’s repo cd to the “205-Using-Security-Group-References/cdk-AWS-Cookbook-205” directory and follow the subsequent steps: 

```
cd 205-Using-Security-Group-References/cdk-AWS-Cookbook-205/
test -d .venv || python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
cdk deploy
```

### TWait for the cdk deploy command to complete. 

### TWe created a helper.py script to let you easily create and export environment variables to make subsequent commands easier. Run the script, and copy the output to your terminal to export variables:

`python helper.py`



## Clean up 
### Detach the security groups you created from each instance and attach the VPC default security group (so that you can delete the security groups in the next step):

```
aws ec2 modify-instance-attribute --instance-id \
$INSTANCE_ID_1 --groups $DEFAULT_VPC_SECURITY_GROUP

aws ec2 modify-instance-attribute --instance-id \
$INSTANCE_ID_2 --groups $DEFAULT_VPC_SECURITY_GROUP
```

### Delete the security group that you created:

```
aws ec2 delete-security-group --group-id $SG_ID
```

### To clean up the environment variables, run the helper.py script in this recipe’s cdk- directory with the --unset flag, and copy the output to your terminal to export variables:

`python helper.py --unset`

### Unset the environment variable that you created manually:

```
unset SG_ID
unset INSTANCE_ID_3
unset INSTANCE_PROFILE
```

### Use the AWS CDK to destroy the resources, deactivate your Python virtual environment, and go to the root of the chapter:

`cdk destroy && deactivate && rm -r .venv/ && cd ../..`

## Hint
```
INSTANCE_ID_3=$(aws ec2 run-instances \
--image-id $AMZNLINUXAMI --count 1 \
--instance-type t3.nano --security-group-ids $SG_ID \
--subnet-id $VPC_ISOLATED_SUBNET_1 \
--output text --query Instances[0].InstanceId)
```

### Retrieve the IAM Instance Profile Arn for Instance2 so that you can associate it with your new instance. This will allow the instance to register with SSM:

```
INSTANCE_PROFILE=$(aws ec2 describe-iam-instance-profile-associations \
--filter "Name=instance-id,Values=$INSTANCE_ID_2" \
--output text --query IamInstanceProfileAssociations[0].IamInstanceProfile.Arn)
```

### Associate the IAM Instance Profile with Instance3:

```
aws ec2 associate-iam-instance-profile \
    --instance-id $INSTANCE_ID_3 \
    --iam-instance-profile Arn=$INSTANCE_PROFILE
```

### Reboot the instance to have it register with SSM:

`aws ec2 reboot-instances --instance-ids $INSTANCE_ID_3`

### Once that is complete you can connect to it using SSM Session Manager:

`aws ssm start-session --target $INSTANCE_ID_3`

## TODO 
- Create a VPC
```
set VPC_ID $(aws ec2 create-vpc --cidr-block 10.10.0.0/16 \
          --tag-specifications \
      'ResourceType=vpc,Tags=[{Key=Name,Value=AWSCookbook201}]' \
          --output text --query Vpc.VpcId --profile yoshida_playground_shintaro_yoshida)
```
- 2.1 
```
aws ec2 describe-vpcs --vpc-ids $VPC_ID --profile yoshida_playground_shintaro_yoshida
aws ec2 associate-vpc-cidr-block \
        --cidr-block 10.11.0.0/16 \
        --vpc-id $VPC_ID --profile yoshida_playground_shintaro_yoshida

aws ec2 create-vpc --cidr-block 10.10.0.0/16 \
    --amazon-provided-ipv6-cidr-block \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=AWSCookbook201-IPv6}]' \
    --profile yoshida_playground_shintaro_yoshida
```

- Create a Route Table 
```
set ROUTE_TABLE_ID $(aws ec2 create-route-table --vpc-id $VPC_ID \
                                                  --tag-specifications \
                                                  'ResourceType=route-table,Tags=[{Key=Name,Value=AWSCookbook202}]' \
                                                  --output text --query RouteTable.RouteTableId --profile yoshida_playground_shintaro_yoshida)
set ROUTE_TABLE_ID_2 $(aws ec2 create-route-table --vpc-id $VPC_ID \
                                                                                                --tag-specifications \
                                                                                                'ResourceType=route-table,Tags=[{Key=Name,Value=AWSCookbook202_2}]' \
                                                                                                --output text --query RouteTable.RouteTableId --profile yoshida_playground_shintaro_yoshida)
set SUBNET_ID_1 $(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.13.2.0/24 --availability-zone {$AWS_REGION}a \
    --tag-specifications \
    'ResourceType=subnet,Tags=[{Key=Name,Value=AWSCookbook202a}]' \
    --output text --query Subnet.SubnetId --profile yoshida_playground_shintaro_yoshida)
set SUBNET_ID_2 $(aws ec2 create-subnet --vpc-id $VPC_ID \
    --cidr-block 10.13.3.0/24 --availability-zone {$AWS_REGION}c \
    --tag-specifications \
    'ResourceType=subnet,Tags=[{Key=Name,Value=AWSCookbook202c}]' \
    --output text --query Subnet.SubnetId --profile yoshida_playground_shintaro_yoshida)
set AWS_REGION ap-northeast-1
aws ec2 describe-availability-zones --region $AWS_REGION
aws ec2 associate-route-table \
    --route-table-id $ROUTE_TABLE_ID --subnet-id $SUBNET_ID_1 --profile yoshida_playground_shintaro_yoshida

aws ec2 associate-route-table \
    --route-table-id $ROUTE_TABLE_ID --subnet-id $SUBNET_ID_2 --profile yoshida_playground_shintaro_yoshida
aws ec2 describe-subnets --subnet-ids $SUBNET_ID_1 --profile yoshida_playground_shintaro_yoshida
                                              aws ec2 describe-subnets --subnet-ids $SUBNET_ID_2 --profile yoshida_playground_shintaro_yoshida
```

## 2.3 
```
set INET_GATEWAY_ID $(aws ec2 create-internet-gateway \
    --tag-specifications \
'ResourceType=internet-gateway,Tags=[{Key=Name,Value=AWSCookbook202}]' \
    --output text --query InternetGateway.InternetGatewayId --profile yoshida_playground_shintaro_yoshida)
aws ec2 attach-internet-gateway \
                                                      --internet-gateway-id $INET_GATEWAY_ID --vpc-id $VPC_ID --profile yoshida_playground_shintaro_yoshida
aws ec2 create-route --route-table-id $ROUTE_TABLE_ID \
                                                      --destination-cidr-block 0.0.0.0/0 --gateway-id $INET_GATEWAY_ID --profile yoshida_playground_shintaro_yoshida
```

- Create a Security Group
```
set SG_ID $(aws ec2 create-security-group \
          --group-name AWSCookbook205Sg \
          --description "Instance Security Group" --vpc-id $VPC_ID \
          --output text --query GroupId --profile yoshida_playground_shintaro_yoshida)
```
- Attach Instance with Security Group 
```
aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID_1 \
    --groups $SG_ID

aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID_2 \
    --groups $SG_ID
```
