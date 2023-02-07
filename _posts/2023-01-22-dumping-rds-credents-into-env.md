---
layout: post
title:  "CDK: Getting RDS credentials from secretsmanager to a .env"
date:   2023-01-22 11:57:04 +0000
categories: CDK, development, .env, secretsmanager
---

When creating an RDS instance using CDK, I generate the credentials using aws_secretsmanager.Secret, like this:

```typescript
const credentials = new aws_secretsmanager.Secret(this, 'RdsCredents', {
    generateSecretString: {
        secretStringTemplate: JSON.stringify({ username: 'admin' }),
        generateStringKey: 'password',
        excludeCharacters: '"@/\\',
    }
});
```

I then pass the credentials to the RDS instance:

```typescript
this.rds = new aws_rds.DatabaseInstance(this, "RDS", {
    engine: aws_rds.DatabaseInstanceEngine.mysql({
        version: aws_rds.MysqlEngineVersion.VER_8_0_31
    }),
    instanceType: aws_ec2.InstanceType.of(aws_ec2.InstanceClass.T2, aws_ec2.InstanceSize.MICRO),
    vpc: this.vpc,
    vpcSubnets: {
        subnetType: aws_ec2.SubnetType.PRIVATE_ISOLATED
    },
    multiAz: false,
    allocatedStorage: 10,
    storageType: aws_rds.StorageType.GP2,
    databaseName: "laravel",
    credentials: { //here they are!
        username: 'admin', 
        password: credentials.secretValueFromJson('password') 
    },
});
```

What I now need to do is get the secret during the CI/CD Pipeline, and put it into a .env file.

I've created a script to do this for me (`touch database_credentials.sh && chmod +x database_credentials.sh`):

```bash
#!/bin/bash
# Get your RDS secrets from AWS Secrets Manager
SECRET_NAME=$1
RDS_CREDENTS=$(aws secretsmanager get-secret-value --secret-id $SECRET_NAME)
echo "SecretString Password:"
echo $(echo $RDS_CREDENTS | jq '.SecretString | fromjson | .password')
echo "SecretString Username:"
echo $(echo $RDS_CREDENTS | jq '.SecretString | fromjson | .username')
```

I then run this script in the pipeline:

```yaml
- name: Get database credentials
  run: |
    ./database_credentials.sh ${{ secrets.SECRET_NAME }}
```

We can then run a pre-deployment script like this:

```typescript
deploymentStage.addPre(new ShellStep('BuildAssets', {
    installCommands: ['npm i -g npm@latest'],
    env: {
    'rds_secret': 'YourDBSecretNameOrArn',
    'environment_secret': 'YourENVSecretNameOrArn'
    },
    commands: [
    './db_credentials.sh $rds_secret',
    './environment_variables.sh $environment_secrets',
    'composer install',
    'npm ci',
    'npm run build',
    ],
}));
```

We now have a .env file full of juicy secrets, ready to be consumed by our application