## PLATFORM

NOTE: the working directory for this sectin is: `sa-ssp-aws/platform/`

In this section you will deploy two Auto Scale Groups (ASGs) of five EC2 servers each: 1 Vault ASG, 1 Consul ASG.

### Secrets Management - VAULT

### 1. Prepare for terraform deployment

With your AWS credentials exported, and the correct information added to your `sa-ssp-aws/platform/vault-ent-aws/terraform.tfvars` from the steps above, you should now be able to run:

```sh
terraform init
terraform plan
```

Unless you received errors from the above commands, you are now ready to deploy the Vault and Consul ASGs with:

```sh
terraform apply
```

Upon successful completion you should see:

```hcl
asg_name = "sa-vault"
kms_key_arn = "arn:aws:kms:us-west-2:652626842611:key/10a8c141-d359-495b-92fc-546fa00ff109"
launch_template_id = "lt-0bb553c46288080c6"
vault_lb_arn = "arn:aws:elasticloadbalancing:us-west-2:652626842611:loadbalancer/app/sa-vault-lb/1e4797b210ca679f"
vault_lb_dns_name = "internal-sa-vault-lb-1711292900.us-west-2.elb.amazonaws.com"
vault_lb_zone_id = "Z1H1FL5HABSF5"
vault_sg_id = "sg-0528380228b1666ce"
vault_target_group_arn = "arn:aws:elasticloadbalancing:us-west-2:652626842611:targetgroup/sa-vault-tg/fbfde0da72d05fcf"
```

You will need the `vault_lb_dns_name` value in the following steps.

### 2. Verify Vault Scale Group

Using the AWS 'Secure Session Manager' (`aws ssm`) command, connect to a Vault instance and verify the Vault Cluster is running and healthy.

Using the AWS Auto Scaling Group (ASG) name in the above terraform output, get the `instance id` of an EC2 Scale Group member:

```sh
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names `terraform output -raw asg_name` --no-cli-pager --query "AutoScalingGroups[*].Instances[*].InstanceId"
```

Select an instance ID from the list.

```sh
aws ssm start-session --target <instance_id>
```

Export the following two variables so that you can interact with Vault:
```sh
export VAULT_ADDR="https://127.0.0.1:8200"
export VAULT_CACERT="/opt/vault/tls/vault-ca.pem"
```

Verify Vault is running with:
```sh
vault status
```

**NOTE:** Vault is currently: `Sealed: true`

### 3. Initialize Vault Cluster

//TODO: Do I need to unseal every instance?

```sh
vault operator init
```

**NOTE** Copy the `Recovery Keys` and the `Initial Root Token` for future steps.

```sh
export VAULT_TOKEN=<initial_root_token>
```

Unseal Vault:
```sh
vault operator unseal
```

You will be presented with the following prompt:

```sh
Unseal Key (will be hidden):
```

Enter the value of `Recovery Key 1:` retrieved from the `vault operator init` command above.

You should see an output as such:
```sh
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.12.2+ent
Build Date               2022-11-23T21:33:30Z
Storage Type             raft
Cluster Name             vault-cluster-8a6b623b
Cluster ID               1e4d4bf9-7dce-44ff-126e-0526f3455157
HA Enabled               true
HA Cluster               https://10.0.101.40:8201
HA Mode                  active
Active Since             2023-01-11T18:07:07.686021267Z
Raft Committed Index     137
Raft Applied Index       137
Last WAL                 27
```

**NOTE:** Vault is now: `Sealed: false` and ready for use.

//FIXME: exit the vault `aws ssm` session. How do I get local vault
```sh
vault operator raft list-peers
```

You should see something like:
```sh
Node                   Address              State       Voter
----                   -------              -----       -----
i-039ae7e6ddeb59b4d    10.0.102.121:8201    leader      true
i-0e238e7297f0d2b52    10.0.101.145:8201    follower    true
i-0d5546d3eeb23df85    10.0.103.55:8201     follower    true
i-0560234cb583fb773    10.0.103.35:8201     follower    true
i-0a403b210f7af110a    10.0.102.179:8201    follower    true
```

**NOTE:** Repeat this to unseal each Vault instance in the scale group



### 3. Configure Vault for Consul Gossip Key
//TODO: **YOU ARE HERE** Everything above works.

https://developer.hashicorp.com/consul/tutorials/vault-secure/vault-pki-consul-secure-tls

### n. Enable Vault Secrets Engine

```sh
vault secrets enable -path=consul kv-v2
```


Copy the local `../../inputs/consul.hclic` to the Vault ASG instance:

```sh
cd ~
mkdir tmp
vi consul.hclic
```

Paste the contents of your locally saved Consul license located: `../../inputs/consul.hclic`

Store Consul license in Vault:
```sh
vault kv put consul/secret/enterpriselicense key="$(cat ./consul.hclic)"
```

You should see a response resembling:

```sh
============ Secret Path ============
consul/data/secret/enterpriselicense

======= Metadata =======
Key                Value
---                -----
created_time       2023-01-11T20:03:29.449280648Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```


Store Consul Gossip Key in Vault - substituting the Consul Gossip Key generated earlier:

```sh
vault kv put consul/secret/gossip gossip="<consul_gossip_key>"
```

For example: `vault kv put consul/secret/gossip gossip="mpO9YcSq+YnOqK2Prd0igm2kQObneGCjspOfi7JSH70="`

The respose should resemble:

```sh
====== Secret Path ======
consul/data/secret/gossip

======= Metadata =======
Key                Value
---                -----
created_time       2023-01-11T20:06:47.450963339Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```


### n. Configure Vault for Consul mTLS Cert Management

https://developer.hashicorp.com/consul/tutorials/vault-secure/vault-pki-consul-secure-tls


Setup PKI secrets engine:
```sh
vault secrets enable pki
```

//TODO: why is the example setting 10 year certs? WHY???

NOTE: "dc1.consul" is: `<consul_dc>.<consul_tld>`

```sh
vault secrets tune -max-lease-ttl=87600h pki
vault write -field=certificate pki/root/generate/internal \
    common_name="dc1.consul" \
    ttl=87600h | tee consul_ca.crt
```

Create a Vault role for the consul server:

```sh
vault write pki/roles/consul-server \
    allowed_domains="dc1.consul,consul-server,consul-server.consul,consul-server.consul.svc" \
    allow_subdomains=true \
    allow_bare_domains=true \
    allow_localhost=true \
    generate_lease=true \
    max_ttl="720h"
```

//TODO: this error comes back from `generate_lease`, why?:
```
WARNING! The following warnings were returned from Vault:

  * it is encouraged to disable generate_lease and rely on PKI's native
  capabilities when possible; this option can cause Vault-wide issues with
  large numbers of issued certificates
```


```sh
vault secrets enable -path connect-root pki
```


### n. Enable k8s Auth

```sh
vault auth enable kubernetes
```

**NOTE:** on your local machine! 
//TODO: move all this (The platform section) to a bastian host in platsvcs vpc


```sh
export token_reviewer_jwt=$(kubectl get secret \
  $(kubectl get serviceaccount vault -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{ .data.token }' | base64 --decode)
```

```sh
export kubernetes_ca_cert=$(kubectl get secret \
  $(kubectl get serviceaccount vault -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{ .data.ca\.crt }' | base64 --decode)
```

```sh
export kubernetes_host_url=$(kubectl config view --raw --minify --flatten \
  -o jsonpath='{.clusters[].cluster.server}')
```


```sh
vault write auth/kubernetes/config \
  token_reviewer_jwt="${token_reviewer_jwt}" \
  kubernetes_host="${kubernetes_host_url}" \
  kubernetes_ca_cert="${kubernetes_ca_cert}"
```

```sh
vault read auth/kubernetes/config
```

This response should resemble:

```sh
Key                       Value
---                       -----
disable_iss_validation    true
disable_local_ca_jwt      false
issuer                    n/a
kubernetes_ca_cert        -----BEGIN CERTIFICATE-----
MIIC/jCCAeagAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTIzMDExMTE3MjU1OFoXDTMzMDEwODE3MjU1OFowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALxN
4D97jnYvej6PPUBsnQE2L+z2x0picKztDLIeQazn+lxNekSRRGxrlPHDHsuIluBc
z2JKGtFqYKwIYx+/4yw6pHkCgR17v3sJHhUBkQTR/LO5tq4t5u3ukPc5qPflil+D
OZIXDl6tz4Fcy5DjjXuGPllPW+L4m3+tE9X06GVFrMAu9SiHtdCPFqYRtdH1qhbA
F35K6LsfUa7z7vyEBQVFYfIkY++XVY+Hsj3bsjLIY8ZkZcqArhThuMnIWPVgGiXR
OeQR8RR8xfBGqkXt9olALVumM3EJ79BiB3diXWSMUOu/tqdjBwlPtho1qwkgp7Fv
4iKerzOp9Q9wFbagT9UCAwEAAaNZMFcwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
/wQFMAMBAf8wHQYDVR0OBBYEFFq72DjUZQha7APNJ5ZZ85ezhzXhMBUGA1UdEQQO
MAyCCmt1YmVybmV0ZXMwDQYJKoZIhvcNAQELBQADggEBAIlkgJrJVlScDi32vOdc
JRFDlUComUtovtTNBGkI2uH0ygufpohj0FT0AsjNOswg+kRXbOZU+Wy/R8j3Pdts
+lcAR25K2ePACHwoZdtL8Q1a1byQ6tV5TMZOiUonj1uR5u6gwwZMngUXDqNBbJYY
E0wFQ3QcfPaE29YUwk1OJywslLX9qinANFlbi2JBqp6045qqvp/U8zO8utKGxbhf
p/VHvlFZoXIbuA5LiEm2om6z5KJ3pkMP4Ot5TOjuIHAdzXRLfUej3ARkrhlwRzBc
BoOJWeNR8ZpKuRz8AJ3eafdMoXgdhri0GCqzr5eLmbTDK0Ma9yiz9Zsaam72QH+U
NoY=
-----END CERTIFICATE-----
kubernetes_host           https://11A21DB9B12B13E5706CC9AD9CCD7187.gr7.us-west-2.eks.amazonaws.com
pem_keys                  []
```


**NEXT** Create Policies:
https://developer.hashicorp.com/consul/tutorials/vault-secure/kubernetes-vault-consul-secrets-management#generate-vault-policies






### n. Install the Vault Injector into k8s:

Disconnect from your AWS SSM session (don't run this in the Vault ASG instance):

```sh
export VAULT_PRIVATE_ADDR=`terraform output -raw vault_lb_dns_name`
cat > vault-values.yaml << EOF
injector:
  enabled: true
  externalVaultAddr: "https://${VAULT_PRIVATE_ADDR}:8200"
EOF
```

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update
#helm install vault -f ./vault-values.yaml hashicorp/vault --version "0.20.0"
helm install vault -f ./vault-values.yaml hashicorp/vault
```


### Secure Communications - CONSUL