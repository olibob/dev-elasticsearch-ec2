Deploy a throwaway one node elasticsearch server on EC2 for dev/testing purposes with basic authentication (based on Centos7).

# Prerequisites

SSH client, Ansible, AWS CLI.

# AWS

Create a stack with the `aws.yaml`tempalate in the `AWS_Elasticsearch_EC2_Instance` directory.

Upload the template to Cloudformation via the AWS console and create the stack interactively.

```
aws cloudformation create-stack --stack-name Dev-Els  \
    --template-body file://AWS_Elasticsearch_EC2_Instance/aws.yaml \
    --parameters file://AWS_Elasticsearch_EC2_Instance/params.json \
    --capabilities CAPABILITY_IAM \
    --profile <your profile here>
```

If you want to use the cli, create your own parameters file named `params.json` in the `AWS_Elasticsearch_EC2_Instance` directory.

Here's an example `params.json` file for reference:

```
[
  {
    "ParameterKey": "VPCId",
    "ParameterValue": "vpc-b0ed07d5"
  }, 
  {
    "ParameterKey": "VPCSubnet",
    "ParameterValue": "subnet-d86ea1bd"
  },
  {
    "ParameterKey": "SSHAccessCIDR",
    "ParameterValue": "194.168.2.1/24"
  },
  {
    "ParameterKey": "HTTPAccessCIDR",
    "ParameterValue": "0.0.0.0/0"
  },
  {
    "ParameterKey": "KeyPairName",
    "ParameterValue": "MyKeyPair"
  },
  {
    "ParameterKey": "InstanceAMI",
    "ParameterValue": "ami-6e28b517"
  },
  {
    "ParameterKey": "ElsInstanceType",
    "ParameterValue": "t2.medium"
  }
]
```

Monitor your stack creation (I use jq here, you can use the build in `--query` as well)

```
aws cloudformation describe-stacks --stack-name  Dev-Els --profile <your profile> | jq -r '.Stacks[].StackStatus'
```

Get the outputs from the stack (Instance DNS name and public IP)

```
aws cloudformation describe-stacks --stack-name  Dev-Els --profile <your profile> | jq -r '.Stacks[].Outputs'
``` 

Delete stack

```
aws cloudformation delete-stack --stack-name Dev-Els --profile <your profile>
```

After deletion is complete, if you query again, you get a "good" error: "An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id Dev-Els does not exist".

# Ansible playbook

## Configuration

### Inventory

Once your stack is created, recover the stack outputs and add the target hostname to the `ec2.txt` inventory file.

### Certificates

Create an `sslcerts` directory at the root of the current directory and add your:

- fullchain certificate: name it `ssl-bundle.crt`
- private key: name it `private.key.pem`

This is needed for the TLS configuration of the nginx reverse proxy. 

### Playbook

Edit the `vars`section of the playbook and set:

- the elasticsearch and kibana package version to install.
- the username for basic authentication
- the password for basic authentication
- the AWS Region

## Run

```
ansible-playbook -i ec2.txt elasticsearch-playbook.yaml 
```

This will setup:

- Elasticsearch
- Kibana
- Nginx reverse proxy
  - elasticsearch vhost
  - kibana vhost
  - htpasswd for basic auth (via SSL)
- awslogs:
  - awslogs.conf to log elasticsearch logs to cloudwatch
- exclude els and kibana from updates if you keep it running for longer than anticipated ;-)