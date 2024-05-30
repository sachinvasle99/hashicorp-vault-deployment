# Install the Vault Helm chart
Create a file named helm-vault-raft-values.yml with the following contents:
```
$ cat > helm-vault-raft-values.yml <<EOF
server:
   affinity: ""
   ha:
      enabled: true
      raft:
         enabled: true
         setNodeId: true
         config: |
            cluster_name = "vault-integrated-storage"
            storage "raft" {
               path    = "/vault/data/"
            }
            listener "tcp" {
               address = "[::]:8200"
               cluster_address = "[::]:8201"
               tls_disable = "true"
            }
            service_registration "kubernetes" {}
EOF
```
Install the latest version of the Vault Helm chart with Integrated Storage.
```
$ cd hashicorp-vault-deployment
$ helm install vault ./vault-helmchart -f helm-vault-raft-values.yml
```
This creates three Vault server instances with an Integrated Storage (Raft) backend.

Along with the Vault pods and Vault Agent Injector pod are deployed in the default namespace.

Get all the pods within the default namespace.
```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          30s
vault-1                                 0/1     Running   0          30s
vault-2                                 0/1     Running   0          30s
vault-agent-injector-56bf46695f-crqqn   1/1     Running   0          30s
```
The vault-0, vault-1, and vault-2 pods deployed run a Vault server and report that they are Running but that they are not ready (0/1). This is because the status check defined in a readinessProbe returns a non-zero exit code.

The vault-agent-injector pod deployed is a Kubernetes Mutation Webhook Controller. The controller intercepts pod events and applies mutations to the pod if specific annotations exist within the request.

Retrieve the status of Vault on the vault-0 pod.
```
$ kubectl exec vault-0 -- vault status
```
Example output:
The status command reports that Vault is not initialized and that it is sealed. For Vault to authenticate with Kubernetes and manage secrets requires that that is initialized and unsealed.
```
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.10.3
Storage Type       raft
HA Enabled         true
command terminated with exit code 2
```
# Initialize and unseal one Vault pod
Vault starts uninitialized and in the sealed state. Prior to initialization the Integrated Storage backend is not prepared to receive data.

Initialize Vault with one key share and one key threshold.
```
$ kubectl exec vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json
```
The operator init command generates a root key that it disassembles into key shares -key-shares=1 and then sets the number of key shares required to unseal Vault -key-threshold=1. These key shares are written to the output as unseal keys in JSON format -format=json. Here the output is redirected to a file named cluster-keys.json.

Display the unseal key found in cluster-keys.json. 
```
$ cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
rrUtT32GztRy/pVWmcH0ZQLCCXon/TxCgi40FL1Zzus=
```
Create a variable named VAULT_UNSEAL_KEY to capture the Vault unseal key.
```
$ VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```
After initialization, Vault is configured to know where and how to access the storage, but does not know how to decrypt any of it. Unsealing is the process of constructing the root key necessary to read the decryption key to decrypt the data, allowing access to the Vault.

Unseal Vault running on the vault-0 pod.
```
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```
The operator unseal command reports that Vault is initialized and unsealed.

Retrieve the status of Vault on the vault-0 pod.
```
$ kubectl exec vault-0 -- vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.10.3
Storage Type            raft
Cluster Name            vault-cluster-16efc511
Cluster ID              649c814a-a505-421d-e4bb-d9175c7e6b38
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2022-05-19T17:41:07.226862254Z
Raft Committed Index    36
Raft Applied Index      36
```

The Vault server is initialized and unsealed.

## Join the other Vaults to the Vault cluster
The Vault server running on the vault-0 pod is a Vault HA cluster with a single node. To display the list of nodes requires that you are logging in with the root token.

Display the root token found in cluster-keys.json.
```
$ cat cluster-keys.json | jq -r ".root_token"
hvs.3VYhJODbhlQPeW5zspVvBCzD
```
Create a variable named CLUSTER_ROOT_TOKEN to capture the Vault unseal key.
```
CLUSTER_ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
```
Login with the root token on the vault-0 pod.
```
$ kubectl exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.
Key                  Value
---                  -----
token                hvs.3VYhJODbhlQPeW5zspVvBCzD
token_accessor       5sy3tZm3qCQ1ai7wTDOS97XG
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

List all the nodes within the Vault cluster for the vault-0 pod.
```
$ kubectl exec vault-0 -- vault operator raft list-peers
Node                                    Address                        State     Voter
----                                    -------                        -----     -----
09d9b35d-0336-7de7-cc94-90a1f3a0aff8    vault-0.vault-internal:8201    leader    true
```
This displays the one node within the Vault cluster. This cluster is addressable through the Kubernetes service vault-0.vault-internal created by the Helm chart. The Vault servers on the other pods need to join this cluster and be unsealed.

Join the Vault server on vault-1 to the Vault cluster.
```
$ kubectl exec vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
Key       Value
---       -----
Joined    true
```
This Vault server joins the cluster sealed. To unseal the Vault server requires the same unseal key, VAULT_UNSEAL_KEY, provided to the first Vault server.

Unseal the Vault server on vault-1 with the unseal key.
```
$ kubectl exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.10.3
Storage Type            raft
Cluster Name            vault-cluster-16efc511
Cluster ID              649c814a-a505-421d-e4bb-d9175c7e6b38
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 standby
Active Node Address     http://192.168.58.131:8200
Raft Committed Index    76
Raft Applied Index      76
```
The Vault server on vault-1 is now a functional node within the Vault cluster.

Join the Vault server on vault-2 to the Vault cluster.
```
$ kubectl exec vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
Key       Value
---       -----
Joined    true
```
Unseal the Vault server on vault-2 with the unseal key.

```
$ kubectl exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.10.3
Storage Type            raft
Cluster Name            vault-cluster-16efc511
Cluster ID              649c814a-a505-421d-e4bb-d9175c7e6b38
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 standby
Active Node Address     http://192.168.58.131:8200
Raft Committed Index    76
Raft Applied Index      76
```

The Vault server on vault-2 is now a functional node within the Vault cluster.

List all the nodes within the Vault cluster for the vault-0 pod.
```
$ kubectl exec vault-0 -- vault operator raft list-peers
Node                                    Address                        State       Voter
----                                    -------                        -----       -----
09d9b35d-0336-7de7-cc94-90a1f3a0aff8    vault-0.vault-internal:8201    leader      true
7078a8b7-7948-c224-a97f-af64771ad999    vault-1.vault-internal:8201    follower    true
aaf46893-0a93-17ce-115e-f57033d7f41d    vault-2.vault-internal:8201    follower    true
```

This displays all three nodes within the Vault cluster.

Get all the pods within the default namespace.
```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          5m49s
vault-1                                 1/1     Running   0          5m48s
vault-2                                 1/1     Running   0          5m47s
vault-agent-injector-5945fb98b5-vzbqv   1/1     Running   0          5m50s
```
The vault-0, vault-1, and vault-2 pods report that they are Running and ready (1/1).

# Create a Vault database role
The web application that you deploy in the Launch a web application section, expects Vault to store a username and password at the path secret/webapp/config. To create this secret requires you to login with the root token, enable the key-value secret engine, and store a secret username and password at that defined path.

Enable database secrets at the path database.
```
$ kubectl exec vault-0 -- vault secrets enable database
Success! Enabled the database secrets engine at: database/
```
Configure the database secrets engine with the connection credentials for the MySQL database.
```
$ kubectl exec vault-0 -- vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql.default.svc.cluster.local:3306)/" \
    allowed_roles="readonly" \
    username="root" \
    password="$ROOT_PASSWORD"
```
Create a database secrets engine role named readonly.
```
$ kubectl exec vault-0 -- vault write database/roles/readonly \
    db_name=mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```
The readonly role generates credentials that are able to perform queries for any table in the database.

Read credentials from the readonly database role.
```
$ kubectl exec vault-0 -- vault read database/creds/readonly
Key                Value
---                -----
lease_id           database/creds/readonly/qtWlgBT1YTQEPKiXe7CrotsT
lease_duration     1h
lease_renewable    true
password           WLESe5T-RLkTj-h-lDbT
username           v-root-readonly-pk168KvLS8sc80Of
```
Vault is able to generate crentials within the MySQL database.

# Configure Kubernetes authentication
The initial root token is a privileged user that can perform any operation at any path. The web application only requires the ability to read secrets defined at a single path. This application should authenticate and be granted a token with limited access.

Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token.

Start an interactive shell session on the vault-0 pod.
```
$ kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
/ $
```
Your system prompt is replaced with a new prompt / $.

Enable the Kubernetes authentication method.
```
$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```
Vault accepts a service token from any client within the Kubernetes cluster. During authentication, Vault verifies that the service account token is valid by querying a token review Kubernetes endpoint.

Configure the Kubernetes authentication method to use the location of the Kubernetes API.
```
$ vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

The environment variable KUBERNETES_PORT_443_TCP_ADDR is defined and references the internal network address of the Kubernetes host.

For a client of the Vault server to read the credentials defined in the Create a Vault database role step requires that the read capability be granted for the path database/creds/readonly.

Write out the policy named devwebapp that enables the read capability for secrets at path database/creds/readonly
```
$ vault policy write devwebapp - <<EOF
path "database/creds/readonly" {
  capabilities = ["read"]
}
EOF
```
Create a Kubernetes authentication role named devweb-app.
```
$ vault write auth/kubernetes/role/devweb-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=devwebapp \
      ttl=24h
```
The role connects a Kubernetes service account, internal-app (created in the next step), and namespace, default, with the Vault policy, devwebapp. The tokens returned after authentication are valid for 24 hours.

Exit the vault-0 pod.
```
$ exit
```


# Install the MySQL Helm chart
MySQL is a fast, reliable, scalable, and easy to use open-source relational database system. MySQL Server is intended for mission-critical, heavy-load production systems as well as for embedding into mass-deployed software.

Add the Bitnami Helm repository.
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```
Install the latest version of the MySQL Helm chart.
```
$ helm install mysql ./mysql-helmchart
```
By default the MySQL Helm chart deploys a single pod a service.

Get all the pods within the default namespace.
```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
mysql-0                                 1/1     Running   0          2m58s
```
Wait until the mysql-0 pod is running and ready (1/1).

The mysql-0 pod runs a MySQL server. a

Get all the services within the default namespace.
```
$  kubectl get services
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
kubernetes                 ClusterIP   10.100.0.1       <none>        443/TCP             3h24m
mysql                      ClusterIP   10.100.68.110    <none>        3306/TCP            15m
mysql-headless             ClusterIP   None             <none>        3306/TCP            15m
```
The mysql service directs request to the mysql-0 pod. Pods within the cluster may address the MySQL server with the address mysql.default.svc.cluster.local.

The MySQL root password is stored as Kubernetes secret. This password is required by Vault to create credentials for the application pod deployed later.

Create a variable named ROOT_PASSWORD that stores the mysql root user password.
```
$ ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
```
The MySQL server, addressed through the service, is ready.

# Launch a web application
The web application pod requires the creation of the internal-app Kubernetes service account specified in the Vault Kubernetes authentication role created in the Configure Kubernetes authentication step.

Define a Kubernetes service account named internal-app.
```
$ cat > internal-app.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app
EOF
```
Create the internal-app service account.

```
$ kubectl apply --filename internal-app.yaml
serviceaccount/internal-app created
```
Define a pod named devwebapp with the web application.
```
$ cat > devwebapp.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-deployment
  template:
    metadata:
      labels:
        app: webapp-deployment
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-cache-enable: "true"
        vault.hashicorp.com/role: "devweb-app"
        vault.hashicorp.com/agent-inject-secret-database-connect.sh: "database/creds/readonly"
        vault.hashicorp.com/agent-inject-template-database-connect.sh: |
          {{- with secret "database/creds/readonly" -}}
          mysql -h mysql.default.svc.cluster.local --user={{ .Data.username }} --password={{ .Data.password }} my_database
          {{- end -}}
    spec:
      serviceAccountName: internal-app
      containers:
        - name: webapp-deployment
          image: jweissig/app:0.0.1
EOF
```
Create the devwebapp deployment.
```
$ kubectl apply --filename devwebapp.yaml
deployment/devwebapp created
```

This definition creates a pod with the specified container running with the internal-app Kubernetes service account. The container within the pod is unaware of the Vault cluster. The Vault Injector service reads the annotations and determines that it should take action vault.hashicorp.com/agent-inject. The credentials, read from Vault at database/creds/readonly, are retrieved by the devwebapp-role Vault role and stored at the file location, /vault/secrets/database-connect.sh, and then mounted on the pod.

The credentials are requested first by the vault-agent-init container to ensure they are present when the application pod initializes. After the application pod initializes, the injector service creates a vault-agent pod that assist the application in maintaining the credentials during initialization. The credentials requested by the vault-agent-init container are cached, vault.hashicorp.com/agent-cache-enable: "true", and used by vault-agent container.

Get all the pods within the default namespace.
```
$ kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
devwebapp                               2/2     Running   0          36s
mysql-0                                 1/1     Running   0          7m32s
vault-0                                 1/1     Running   0          5m40s
vault-1                                 1/1     Running   0          5m40s
vault-2                                 1/1     Running   0          5m40s
vault-agent-injector-76fff8f7c6-lk6gz   1/1     Running   0          5m40s
```

Wait until the devwebapp pod reports that is running and ready (2/2).

Display the secrets written to the file /vault/secrets/database-connect.sh on the devwebapp pod.

```
$ kubectl exec --stdin=true \
    --tty=true devwebapp \
    --container devwebapp \
    -- cat /vault/secrets/database-connect.sh
```
The result displays a mysql command with the credentials generated for this pod.
```
mysql -h mysql.default.svc.cluster.local --user=v-kubernetes-readonly-zpqRzAee2b --password=Jb4epAXSirS2s-pnrI9- my_database
```
## CI/CD Pipeline:
### Consider the deployment pipeline; how will you deploy it?
Example GitHub Actions workflow for deployment:
```
name: Deploy to Minikube

on:
  push:
    branches: [ "main" ] 
  pull_request:
    branches: [ "main" ]

env:
  AWS_ACCOUNT: "058264097424" 
  AWS_REGION: "us-east-1" 
  EKS_CLUSTER_NAME: "demo-app" 
jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}  

    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.20.6'

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --region ${{ secrets.AWS_REGION }} --name ${{ secrets.EKS_CLUSTER_NAME }}

    - name: Deploy to EKS
      run: kubectl apply -f webapp-deployment.yaml   
```

## Authentication Mechanism
### What authentication mechanism are you using? Why?
#### Kubernetes Auth Method: 
Use Vault's Kubernetes auth method, which allows pods to authenticate with Vault using Kubernetes service accounts. This method is secure and scalable, leveraging Kubernetes' native authentication and authorization mechanisms.
Enable the Kubernetes authentication method.
```
$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```
Configure the Kubernetes auth method with your cluster details:
```
$ vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```
Configure the Kubernetes auth method with your cluster details:
```
$ vault write auth/kubernetes/role/devweb-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=devwebapp \
      ttl=24h
```

## Vault Policies for Authorization
### How will you configure and maintain Vault policies for authorization?
1. Policy Creation:
   Define policies in HCL (HashiCorp Configuration Language) or JSON.
   Example policy to allow reading database credentials:
```
   path "secret/data/db-creds" {
  capabilities = ["read"]
   }
```
2. Policy Assignment:
Assign policies to service accounts or tokens, Use the role configuration to bind the policy to the Kubernetes service account.

## Access to Latest Secret
### How can you make sure the application always has access to the latest secret?
The helmchart has deployed a pod named as ```vault-agent-injector``` The Vault Agent templating automatically renews and fetches secrets/tokens, the behavior of how Vault Agent templating does this depends on the type of secret or token. The following is a high level overview of different behaviors.

If a secret or token is renewable, Vault Agent will renew the secret after 2/3 of the secret's lease duration has elapsed.
If a secret or token isn't renewable or leased, Vault Agent will fetch the secret every 5 minutes.

## Handling Dynamic Secrets
### How could your application handle dynamic secrets, e.g., cloud credentials?
Unlike the kv secrets engine which is enabled by default, the AWS secrets engine must be enabled before use. This step is usually done via a configuration management system. 

Example for AWS dynamic secrets:
```
vault secrets enable -path=aws aws
Success! Enabled the aws secrets engine at: aws/
```
The AWS secrets engine is now enabled at aws/. Different secrets engines allow for different behavior. In this case, the AWS secrets engine generates dynamic, on-demand AWS access credentials.

After enabling the AWS secrets engine, you must configure it to authenticate and communicate with AWS. This requires privileged AWS account credentials.

If authenticating with an IAM user, set your AWS Access Key as an environment variable in the terminal that is running your Vault server:

```
$ export AWS_ACCESS_KEY_ID=<aws_access_key_id>
$ export AWS_SECRET_ACCESS_KEY=<aws_secret_key>
```
Your keys must have the IAM permissions listed in the Vault documentation to perform the actions on the rest of this page.

Configure the AWS secrets engine.
```
$ vault write aws/config/root \
    access_key=$AWS_ACCESS_KEY_ID \
    secret_key=$AWS_SECRET_ACCESS_KEY \
    region=us-east-1
```
These credentials are now stored in this AWS secrets engine. The engine will use these credentials when communicating with AWS in future requests.

The next step is to configure a role. Vault knows how to create an IAM user via the AWS API, but it does not know what permissions, groups, and policies you want to attach to that user. This is where roles come in - a role in Vault is a human-friendly identifier to an action.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": ["*"]
    }
  ]
}
```
You need to map this policy document to a named role. To do that, write to aws/roles/:name where :name is your unique name that describes the role (such as aws/roles/my-role):
```
$ vault write aws/roles/my-role \
        credential_type=iam_user \
        policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```
You just told Vault: When I ask for a credential for "my-role", create it and attach the IAM policy { "Version": "2012..." }.

#### Generate the secret

Now that the AWS secrets engine is enabled and configured with a role, you can ask Vault to generate an access key pair for that role by reading from aws/creds/:name, where :name corresponds to the name of an existing role:

```
$ vault read aws/creds/my-role
Key                Value
---                -----
lease_id           aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106e
lease_duration     768h
lease_renewable    true
access_key         AKIAJELUDIANQGRXCTZQ
secret_key         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
security_token     <nil>
```
Success! The access and secret key can now be used to perform any EC2 operations within AWS. Notice that these keys are new, they are not the keys you entered earlier. If you were to run the command a second time, you would get a new access key pair. Each time you read from aws/creds/:name, Vault will connect to AWS and generate a new IAM user and key pair.

Copy the full path of this lease_id value found in the output. This value is used for renewal, revocation, and inspection.

#### Revoke the secret
Vault will automatically revoke this credential after 768 hours (see lease_duration in the output), but perhaps you want to revoke it early. Once the secret is revoked, the access keys are no longer valid.

To revoke the secret, use vault lease revoke with the lease ID that was outputted from vault read when you ran it.
```
$ vault lease revoke aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106
Success! Revoked lease: aws/creds/my-role/0bce0782-32aa-25ec-f61d-c026ff22106e
```
Done! If you login to your AWS account, you will see that no IAM users exist. If you try to use the access keys that were generated, you will find that they no longer work.
