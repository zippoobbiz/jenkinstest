# Building Jenkins pipeline with Vault

This is a simple guide of how to integrate Vault with Jenkins.
You will find three ways of using vault to secure your secrets in this page.

* Vault plugin
* Scripted pipeline
* Declarative pipeline

## Vault Setup

#### start your vault locally

```bash
#!/bin/bash
vault server -dev
```

#### secrets sample

Unseal Key: bQzJlBT5+WY2Cw5WcXyQ1qgkAvz1nazB+3nCTlA4GdQ=

Root Token: s.0h9b29GjxdUSz2vAEidLidz4

#### enable approle

```bash
#!/bin/bash
vault auth enable approle
```

#### create policy

```bash
#!/bin/bash
cat > kv2_admin.hcl << EOF
path "secret/*" {
    capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
vault policy write kv2_admin kv2_admin.hcl
```

#### Or

```bash
#!/bin/bash
echo 'path "secret/*" {
    capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}' | vault policy-write kv2_admin -
```

#### create role

```bash
#!/bin/bash
vault write auth/approle/role/local-jenkins-app \
    secret_id_ttl=60m \
    token_num_uses=10 \
    token_ttl=60m \
    token_max_ttl=120m \
    secret_id_num_uses=40 \
    token_policies="kv2_admin"
```

#### read role_id

```bash
#!/bin/bash
vault read auth/approle/role/local-jenkins-app/role-id
```

#### get the secret_id, may need -force

```bash
#!/bin/bash
vault write -f auth/approle/role/local-jenkins-app/secret-id
```

#### login (optional)

```bash
#!/bin/bash
#vault write auth/approle/login \
#    role_id=<role-id> \
#    secret_id=<secret-id>

vault write auth/approle/login \
    role_id=54e2d8ff-44d8-5c00-05f6-b23734928e1d \
    secret_id=0ba72e3b-e79c-be5d-2118-65fe2f09dc3f
```

copy the role_id and secret_id to jenkins-vault plugin or update the existing credentials. Set Path to approle, ID is optional, Description is the name of the credential (wired I know)

#### put a kv pair in kv2

```bash
#!/bin/bash
vault kv put secret/my-secret key=my-value
vault kv get /secret/my-secret
vault kv get -field=key secret/my-secret
```

## Create a pipeline use the vault plugin

Create a new pipeline and setup the Vault Plugin

* fill in **Vault URL**,
* chose the right **Vault Credential**
* add a vault secret
* Set Path to "secret/my-secret"
* Set Environment Variable to prefered name, we use "testing" in this demo
* Set Key name to "key"

Configure the Build part, chose **Execute shell** and paste the following script to the Command window.

```bash
#!/bin/bash
# $testing contains the real value of our kv vaule, however, it will be masked in the logs, so you will see **** instead of the real vaule. Here we do a hacky way by changing the varible to uppercase to verify that everything works fine.
echo $testing
printf '%s\n' "$testing" | awk '{ print toupper($0) }'
```
Save the pipeline and trigger it. You should see **** as well as **MY_VALUE** in the **Console Output**.

## Scripted pipeline

Create a new pipeline and copy the content of provided scripted-pipeline file into the script window. Don't forget to update the **vaultUrl** and **vaultCredentialId**, save it and trigger the pipeline.

Until this point, you can use the vault plugin gui or scripted-pipeline to get access to the vault secret via the credentials created above, however, vault tokens have ttl, which means the access will be revoked after a certian period of time, in our case **secret-id** will expire in 60 minutes, then you will need to re-generate the secret id and update your credentials in Jenkins. Here is another way of achiving the same goal using the declarative pipeline, which will solve this problem.

## Declarative pipeline

This part we will create a token for jenkins and grant this token access to the approle only.
Then an admin of the team can create or renew this approle, and Jenkins will use this approle to perform certain operations

#### create policy "jenkins" to allow access to local-jenkins-app approle

```bash
#!/bin/bash
echo 'path "auth/approle/role/local-jenkins-app/secret-id" {
  capabilities = ["read","create","update"]
}' | vault policy-write jenkins -
```

#### create token using the above policy

```bash
#!/bin/bash
vault token create -policy=jenkins
```

Save the role-id and token to jenkins credentials (as secret text). Put the jenkinsfile into your repo, setup your repo connection and pull, Jenkins will automatically recognize the jenkinsfile and trigger the declarative pipeline.

The admins can revoke the access to the certain secret of local-jenkins-app by changing the policy, in the following case we change policy to default which doesn't have access to /secret, then the pipeline will get a 403 error.

```bash
#!/bin/bash
vault write auth/approle/role/local-jenkins-app \
    secret_id_ttl=60m \
    token_num_uses=10 \
    token_ttl=60m \
    token_max_ttl=120m \
    secret_id_num_uses=40 \
    token_policies="default"
```
