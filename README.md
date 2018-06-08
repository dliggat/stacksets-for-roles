# stacksets-for-roles

[CloudFormation StackSets][ss] can be used to coordinate the creation of resources across a set of AWS accounts and/or regions. This is coordinated from a central adminstrator account, as shown:

![StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/images/stack_set_conceptual_sv.png)

We can use this to deploy a series of IAM Roles that are used for federated login.

Clone this repo to begin:

```bash
git clone https://github.com/dliggat/stacksets-for-roles
cd stacksets-for-roles
```

## Establishing Trust

We need to set up certain trust relationships in order for this to work.

1. The administrator account needs to trust the CloudFormation _service_, to assume a role that grants permissions in the target account. The service will assume roles in the target accounts in order to create the necessary IAM roles in those accounts.
2. The target accounts need to trust the administrator account. In particular, they need to trust the administrator account to assume a role that will be used to create resources.


### 1. Trusting the CloudFormation Service

We need to deploy the `templates/stackset-config/admin.yaml` template into the Administrator account. It sets up the trust on the service:

```yaml
[...]
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudformation.amazonaws.com
            Action:
              - sts:AssumeRole
[...]
```

To deploy, run the following command (in an appropriate profile for the Administrator account):

```bash
aws cloudformation create-stack --stack-name stackset-admin \
    --template-body file://templates/stackset-config/admin.yaml \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```


### 2. Trusting the Administrator Account

We need to deploy the `templates/stackset-config/target.yaml` template into the target account(s). It sets up the trust on the administrator account:

```yaml
[...]
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Ref AdministratorAccountId
            Action:
              - sts:AssumeRole
[...]
```


To deploy, run the following command (in an appropriate profile for each target account), using the appropriate admin account ID:

```bash
ADMIN_ACCOUNT_ID=111122223333
aws cloudformation create-stack --stack-name stackset-target \
    --template-body file://templates/stackset-config/target.yaml \
    --parameters ParameterKey=AdministratorAccountId,ParameterValue=$ADMIN_ACCOUNT_ID \
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```


[ss]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html
