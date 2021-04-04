+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Encrypted Minio Storage with KMS Setup"
description = "Encrypted Minio Storage with KMS Setup"
date = 2021-02-06T00:00:00+05:30
tags = [
    "kubernetes",
    "minio",
    "KMS",
    "encryption",
    "storage",
    "hashicorp vault",
    "KES",
    "block storage",
    "s3"
]
categories = [
    "storage",
]
series = ["storage"]
aliases = ["storage"]
images = [
    "/images/minio-kes-vault.jpg",
    "/images/app-kes-kms.png"
]
+++

Minio is an S3 compliant data storage service. It can be hosted on premises and even supports distribution across multiple nodes. To meet certain data protection regulations, data is required to be encrypted the moment it is written to disk. Minio supports two types of encryption schemes

- SSE-S3 (Server side encryption) — Encryption key is managed on server side typically using a KMS
- SSE-C (Client side encryption) — Encryption key is managed by clients and provided as request headers to Minio

Goal of this blog is to guide you through setting up Minio with server side encryption. For server side encryption a KMS(key management system) is required. We have chosen Hashicorp vault as KMS here. Minio also supports a Key Encryption Service(KES) which is a stateless cryptographic operations service for Minio with the keys provided from KMS.

KES supports only one form authentication currently which is mTLS. Minio authenticates itself to KES with client TLS certificate for every connection. At a time, KES also supports different policies to be applied for different clients(Eg: Minio). KES acts like a lightweight gateway to use KMS thus abstracting out KMS footprint to clients.

This blogs covers following in step wise manner:

1. Concepts Overview — Some of my exploration notes
2. Generate a root CA certificate pair
3. Hashicorp vault setup
4. Minio KES setup
5. Minio Standalone Setup
6. Minio Distributed Mode Setup
7. Minio Benchmarking using S3 benchmark
8. Additional information

Points 2 through 5 provides information on Minio Standalone setup. Where as Point 6 is required if we want to turn the Minio setup into distributed mode.

Point 7 discusses about benchmarking and how to read the results.

Point 8 is some additional information required especially for Devops or Sys Admins.

> If you want to read the same blog on notion — https://www.notion.so/Published-Encrypted-Minio-Storage-Setup-6ff3e91640b84ba59a6466ac95656503


# 1. Concepts Overview:
### Minio - Kubernetes Native,High Performance Object Storage
- Claims 183GB/s (Read) and 171GB/s (Write) speed on standard hardware
- Can be federated with other minio clusters spanning multiple datacenters
- Native to kubernetes
- MINIO S3 Gateway (Microsoft Azure uses it) and more compatible S3 API
#### Security Overview:
Master Key → generates External Key(EK)(Data Key) → derives Key Encryption Key(KEK) → encrypts Object Encryption Key(OEK) → stored as part of object metadata

OEK is used in Secure channel to encrypt 65536 bytes at a time with unique secret key derived from OEK and with unique nonce ⇒ PRF(OEK, part_no)

- OEK is generated and encrypted with KEK. Encrypted OEK is part of object metadata
- KEK is always derived from PRF(EK, IV, context_values). IV, context_values, Master Key ID and encrypted EK are part of object metadata
- EK is generated from master key from KMS
> Ref: https://docs.min.io/docs/minio-security-overview.html

### Hashicorp Vault
#### Seal/Unseal:
- Shamir seal
- This is the unseal process: the shards are added one at a time (in any order) until enough shards are present to reconstruct the key and decrypt the master key.
#### Lease/Renew/Revoke:
- Vault promises that the data will be valid for the given duration, or Time To Live (TTL).
consumers of secrets need to check in with Vault routinely to either renew the lease (if allowed) or request a replacement secret
- To manually revoke → vault lease revoke
- To renew lease from current time → vault lease renew -increment=3600 my-lease-id
- Revoke with prefix → vault lease revoke -prefix aws/
#### Authentication:
- Client should be authenticated and gets a token with attached policies
- Enabling auth method → vault write sys/auth/my-auth type=userpass
- Follow docs
#### Tokens:
- Tokens maps to policies
- Root tokens are generated in 3 ways — init, using another root token and using generate-root
- Root tokens should be revoked as soon as initial setup/debug/exploration is done
- Generate new root token with key shards → vault operator generate-root
- Child tokens are revoked when parent is revoked, whereas orphan tokens can be created to make sure only that token is revoked but not the children.
- Token accessors are used to renew, revoke, lookup props and caps.
- Tokens can have TTLs except root(TTL=0 is infinite). Tokens TTLs can be renewed with vault token renew
- All Tokens have hard MAX TTL of 32 days. For long living connections use periodic tokens(renew them frequently type). Tokens can be renewed with token store roles, using appRole, have root token with auth/token/create endpoint
#### Policies:
- Refer — https://www.vaultproject.io/docs/concepts/policies
- Policy-Authorization Workflow
- Built in policies — default(attached with every token, cannot be deleted by can be excluded while token creation ) and root(available and cannot be deleted)
- Root tokens are created on initialization and should be revoked before putting in prod
- List policies → vault read sys/policy
- create policy → vault policy write policy-name policy-file.hcl
- update policy → vault write sys/policy/my-existing-policy policy=@updated-policy.json
- delete policy → vault delete sys/policy/policy-name
- Associating tokens with policy on create → vault token create -policy=dev-readonly -policy=logs
#### High availability mode:
- support storage backend to get high HA mode
- By default HA mode is turned. In general, the bottleneck of Vault is the data store itself, not Vault core. For example: to increase the scalability of Vault with Consul, you would generally scale Consul instead of Vault
#### GPG and keybase support:
- On initialisation, the unseal keys generated can be encrypted with GPG public keys of corresponding users and only private key of that user can be used to decrypt it. This is useful when the terminal output is eavesdropped.
- Below command ensures encryption with GPG keys
`vault operator init -key-shares=3 -key-threshold=2 \\ -pgp-keys="jeff.asc,vishal.asc,seth.asc"`
- Unsealing with GPG key → `echo “wcBMA37…” | base64 — decode | gpg -dq`

### KES Server
- Ref: https://blog.min.io/introducing-kes/
- Responsible for cryptographic operations, thus only delegating master key storage to KMS
- Stateless and can scale very well
- Pros — Simple, secure(with TLS avoids username and password)
#### Pre-Requisites for Setup:
- Certificates for KMS and KES

# 2. Generate a Root CA certificate pair
1. Create a ca-config.json CA configuration file
```json
{
  "CN": "MyCompany",
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "server": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
```

2. Create a ca-csr.json self signed certificate signing request file
```json
## Content of ca-csr.json
{
	  "CN": "MyCompany",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names":[{
	    "C": "INDIA",
	    "ST": "MyState",
	    "L": "MyCity",
	    "O": "MyCompany",
	    "OU": "MyUnit"
	  }]
}
```

3. Generate certificates
```bash
$ ./cfssl gencert -initca ca-csr.json | ../cfssljson -bare ca
## Generates 3 files
- ca.pem # public certificate
- ca-key.pem # private key
- ca.csr # certificate signing reque
```

4. You can also configure ca certificate at node level
- copy ca.pem to system trust store
- run update-ca-trust command

# 3. Hashicorp vault setup
Generate TLS certificates to access vault from KES: vault.sample-ns.svc.cluster.local
1. Create server.json file with supported domain names
```json
{
    "CN": "MyCompany",
    "hosts": [
        "vault.sample-ns.svc.cluster.local",
        "vault.sample-ns.pod.cluster.local",
        "vault.sample-ns",
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names":[{
	    "C": "INDIA",
	    "ST": "MyState",
	    "L": "MyCity",
	    "O": "MyCompany",
	    "OU": "MyUnit"
	  }]
}
```

2. Run the command to generate certs
```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
## Generates 3 files
- server.pem # public certificate
- server-key.pem # private certificate
- server.csr # certificate signing request
```

#### Create vault-config.json:
```json
{
   "api_addr": "<https://0.0.0.0:8200>",
   "backend": {
     "file": {
       "path": "file"
     }
   },
  "default_lease_ttl": "168h",
  "max_lease_ttl": "720h",
  "listener": {
    "tcp": {
      "address": "0.0.0.0:8200",
      "tls_cert_file": "server.pem",
      "tls_key_file": "server-key.pem",
      "tls_min_version": "tls12"
    }
  }
}
```

#### Start vault:
```bash
# Starts up vault server on https://0.0.0.0:8200
./vault server -config vault-config.json
```

#### Configure Vault for KES:
1. Create a kes-policy.hcl policy file for KES

```yaml
path "kv/*" {
     capabilities = [ "create", "read", "delete" ]
}
```
2. Initialise Vault
```bash
$ export VAULT_SKIP_VERIFY=true
##########################
$ ./vault operator init
Unseal Key 1: JGDVZdF1f9UKYxYvIiW/tgfB24StDcTevJzzWX0K+TAj
Unseal Key 2: USjqCkON4/gn/oX2PzU1iz6STLuBohf14aP0GpzP+rge
Unseal Key 3: JEX0IJMOEMMwX2Fbld3IwkNLd1OqIU3OUQfSj7YQkfcL
Unseal Key 4: TzxJ5GJuHn4sA4NQDr/BWNKsXhBtO5FjqfJoV48cbYes
Unseal Key 5: bz+kUh5iw/Fakz9aUVP1UWJ+VvQDC3tLrjiSZ8LFkdgg
Initial Root Token: s.CG8JQskBbOVz43Vn9pvE7bgq
Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.
Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!
It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
##########################
$ ./vault operator unseal JGDVZdF1f9UKYxYvIiW/tgfB24StDcTevJzzWX0K+TAj && ./vault operator unseal JEX0IJMOEMMwX2Fbld3IwkNLd1OqIU3OUQfSj7YQkfcL && ./vault operator unseal TzxJ5GJuHn4sA4NQDr/BWNKsXhBtO5FjqfJoV48cbYes
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       675f42c6-978d-85bb-10db-c6fb1f591743
Version            1.5.4
HA Enabled         false
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       675f42c6-978d-85bb-10db-c6fb1f591743
Version            1.5.4
HA Enabled         false
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         1.5.4
Cluster Name    vault-cluster-d954d2ef
Cluster ID      81d91296-59dd-4048-0969-309439d16b2e
HA Enabled      false
##########################
$ export VAULT_TOKEN=s.CG8JQskBbOVz43Vn9pvE7bgq
$ export VAULT_SKIP_VERIFY=true
$ ./vault secrets enable kv
# approle allows authentication and binds to a policy
$ ./vault auth enable approle
$ ./vault policy write kes-policy ./kes-policy.hcl
$ ./vault write auth/approle/role/kes-role token_num_uses=0  secret_id_num_uses=0  period=5m
$ ./vault write auth/approle/role/kes-role policies=kes-policy
$ ./vault read auth/approle/role/kes-role/role-id
$ ./vault write -f auth/approle/role/kes-role/secret-id
```

# 4. Minio KES setup
{{< figure src="/images/app-kes-kms.png" >}}
#### Generate TLS certificate to access KES from Minio:
1. Create server.json file with supported domain names
```json
{
    "CN": "MyCompany",
    "hosts": [
        "minio-kes.sample-ns.svc.cluster.local",
        "minio-kes.sample-ns.pod.cluster.local",
        "minio-kes.sample-ns",
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names":[{
	    "C": "INDIA",
	    "ST": "MyState",
	    "L": "MyCity",
	    "O": "MyCompany",
	    "OU": "MyUnit"
	  }]
}
```
2. Run the command to generate certs:
```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
## Generates 3 files
- server.pem # public certificate
- server-key.pem # private certificate
- server.csr # certificate signing request
```

#### Generate app identity for Minio using KES client:
```bash
$ ./kes tool identity new --key=app.key --cert=app.cert app
$ ./kes tool identity of app.cert
$ export APP_IDENTITY=$(./kes tool identity of app.cert)
```

#### Create server-config.yml:
```yaml
address: 0.0.0.0:7373
root:    disabled  # We disable the root identity since we don't need it in this guide
tls:
  key:  server-key.pem
  cert: server.pem
policy:
  my-app:
    paths:
    - /v1/key/create/my-app*
    - /v1/key/generate/my-app*
    - /v1/key/decrypt/my-app*
    identities:
    - ${APP_IDENTITY}
keys:
  vault:
    endpoint: <https://vault.sample-ns:8200>
    approle:
      id:     "70bd11f5-2104-e4c0-8dbc-48f9066218ce" # Your AppRole ID
      secret: "3a06337c-0163-4c82-e81f-9ec9dce9cf5b" # Your AppRole Secret ID
      retry:  15s
    status:
      ping: 10s
    tls:
      ca: ca.pem # Since we use self-signed certificates
```

#### Start KES:
```bash
$ ./kes server --config=server-config.yml --auth=off
```

#### Verify KES to Vault interaction:
```bash
$ export KES_SERVER=https://minio-kes.sample-ns.svc.cluster.local:7373
$ export KES_CLIENT_CERT=app.cert
$ export KES_CLIENT_KEY=app.key
$ ./kes key create my-app-key -k
$ ./vault kv list kv
$ ./kes key derive my-app-key -k
$ ./vault kv get kv/my-app-key
```

# 5. Minio Standalone Setup
#### Minio Startup configuration:

```bash
$ export MINIO_KMS_KES_ENDPOINT=https://minio-kes.sample-ns
svc.cluster.local:7373
$ export MINIO_KMS_KES_KEY_FILE=app.key
$ export MINIO_KMS_KES_CERT_FILE=app.cert
$ export MINIO_KMS_KES_KEY_NAME=my-app-key
$ MINIO_ACCESS_KEY=minioadmin MINIO_SECRET_KEY=minioadmin ./minio server minio-data
# Place CA certs for minio communication to external services
$ cp ca.pem ~/.minio/certs/CAs/
```

#### Minio Client(mc):
```bash
$ ./mc alias set myminio [<http://127.0.0.1:9000>](<http://127.0.0.1:9000/>) minioadmin minioadmin
# Set policy to a bucket
$ ./mc policy set upload myminio/test
# Setup auto encryption on bucket
$ ./mc encrypt set sse-s3 myminio/test
# verify if encryption is enabled on a bucket
$ ./mc encrypt info myminio/bucket/
```

# 6. Minio Distributed Mode Setu
> Generated template from https://min.io/download#/kubernetes

Below configuration is for 4 nodes with four drive paths(1 path on each no)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app: minio
spec:
  clusterIP: None
  ports:
    - port: 9000
      name: minio
  selector:
    app: minio
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
	namespace: sample-ns
  name: minio
spec:
  serviceName: minio
  replicas: 4
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        env:
        - name: MINIO_ACCESS_KEY
          value: "minioadmin"
        - name: MINIO_SECRET_KEY
          value: "minioadmin"
        image: minio/minio
        args:
        - server
        - http://minio-{0...3}.minio.sample-ns.svc.cluster.local/data
        ports:
        - containerPort: 9000
        # These volume mounts are persistent. Each pod in the PetSet
        # gets a volume mounted based on this field.
        volumeMounts:
        - name: data
          mountPath: /data
  # These are converted to volume claims by the controller
  # and mounted at the paths mentioned above.
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      # Uncomment and add storageClass specific to your requirements below. Read more <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1>
      #storageClassName:
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  type: LoadBalancer
  ports:
    - port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    app: minio
```

- Observations of distributed erasure encoding tolerance to failures:
- If a file is deleted in more than N/2 nodes from a bucket, file is not recovered, otherwise tolerable until N/2 nodes.
- If a bucket is deleted in one node, request going to that replica node shows bucket is missing. Requests going through all other replica nodes, the bucket is visible.
- Looks like buckets are recoverable by directly copying missing buckets from other nodes
- Automatic healing happens once in 30 days. Manual healing is getting deprecated soon, hopefully we get knobs for automatic healing before that.
- If mount point is removed, Error: unformatted disk found is seen
- If mount point is restored, it is not recognised by the application, needed a restart of the pod. “Delete pod and wait for to scale back”

#### Storage class setup:
Storage classes “STANDARD” and “REDUCED_REDUNDANCY” can be set to minio or can also be sent from application using “x-amz-storage-class” or using “mc admin config”

Ref: https://github.com/minio/minio/tree/master/docs/erasure/storage-class

#### Issue with distributed mode:
https://github.com/minio/minio/issues/9052

# 7. Minio Benchmarking using S3 benchmark
### Setup
- 4 node cluster — one drive each(5GB PV)
- Node spec — 12 core 16GB RAM

### Observations
#### Scenario 1
- Encryption enabled
- 100KB average object size

##### Run1:
`$ ./s3-benchmark -a minioadmin -b new-test-bucket -s minioadmin -t 10 -u http://0.0.0.0:30100 -z 100K`

```bash
Wasabi benchmark program v2.0
Parameters: url=http://0.0.0.0:30100, bucket=new-test-bucket, region=us-east-1, duration=60, threads=10, loops=1, size=100K
2020/10/29 11:45:42 WARNING: createBucket new-test-bucket error, ignoring BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded and you already own it.
status code: 409, request id: 164261F9CED6AF49, host id:
**Loop 1: PUT time 60.0 secs, objects = 17057, speed = 27.7MB/sec, 284.1 operations/sec. Slowdowns = 0**
**Loop 1: GET time 60.0 secs, objects = 43592, speed = 70.9MB/sec, 726.4 operations/sec. Slowdowns = 0**
**Loop 1: DELETE time 18.3 secs, 933.5 deletes/sec. Slowdowns = 0**
```

##### Run2:
`./s3-benchmark -a minioadmin -b new-test-bucket -s minioadmin -t 10 -u http://0.0.0.0:30100 -z 100K`

```bash
Wasabi benchmark program v2.0
Parameters: url=http://0.0.0.0:30100, bucket=new-test-bucket, region=us-east-1, duration=60, threads=10, loops=1, size=100K
2020/10/29 11:54:41 WARNING: createBucket new-test-bucket error, ignoring BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded and you already own it.
status code: 409, request id: 16426277335B8DBB, host id:
**Loop 1: PUT time 60.0 secs, objects = 17507, speed = 28.5MB/sec, 291.7 operations/sec. Slowdowns = 0**
**Loop 1: GET time 60.0 secs, objects = 43073, speed = 70.1MB/sec, 717.8 operations/sec. Slowdowns = 0**
**Loop 1: DELETE time 18.3 secs, 958.3 deletes/sec. Slowdowns = 0**
```

#### Scenario 2:
- Encryption disabled
- 100KB average object size

##### Run1:
`./s3-benchmark -a minioadmin -b new-unencrypted-test-bucket -s minioadmin -t 10 -u http://0.0.0.0:30100 -z 100K`

```bash
Wasabi benchmark program v2.0
Parameters: url=http://0.0.0.0:30100, bucket=new-unencrypted-test-bucket, region=us-east-1, duration=60, threads=10, loops=1, size=100K
2020/10/29 15:12:54 WARNING: createBucket new-unencrypted-test-bucket error, ignoring BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded and you already own it.
status code: 409, request id: 16426D483783E4E3, host id:
**Loop 1: PUT time 60.0 secs, objects = 18746, speed = 30.5MB/sec, 312.4 operations/sec. Slowdowns = 0**
**Loop 1: GET time 60.0 secs, objects = 59048, speed = 96.1MB/sec, 984.0 operations/sec. Slowdowns = 0**
**Loop 1: DELETE time 20.0 secs, 937.8 deletes/sec. Slowdowns = 0**
```

##### Run2:
`./s3-benchmark -a minioadmin -b new-unencrypted-test-bucket -s minioadmin -t 10 -u http://0.0.0.0:30100 -z 100K`

```bash
Wasabi benchmark program v2.0
Parameters: url=http://0.0.0.0:30100, bucket=new-unencrypted-test-bucket, region=us-east-1, duration=60, threads=10, loops=1, size=100K
2020/10/29 15:16:02 WARNING: createBucket new-unencrypted-test-bucket error, ignoring BucketAlreadyOwnedByYou: Your previous request to create the named bucket succeeded and you already own it.
status code: 409, request id: 16426D73F07F40A9, host id:
**Loop 1: PUT time 60.0 secs, objects = 19272, speed = 31.4MB/sec, 321.0 operations/sec. Slowdowns = 0**
**Loop 1: GET time 60.0 secs, objects = 52027, speed = 84.7MB/sec, 867.0 operations/sec. Slowdowns = 0**
**Loop 1: DELETE time 20.4 secs, 943.5 deletes/sec. Slowdowns = 0**
```

### How do I read benchmarks?
> Loop 1: PUT time 60.0 secs, objects = 17057, speed = 27.7MB/sec, 284.1 operations/sec. Slowdowns = 0
In 60 seconds, 17057 objects whose average size is 100KB is uploaded to minio. Speed of upload is coming around 27.7MB/sec which is average bandwidth utilised during this operation. Also an average of 284.1 files uploaded per second.

> Loop 1: GET time 60.0 secs, objects = 43592, speed = 70.9MB/sec, 726.4 operations/sec. Slowdowns = 0
In 60 seconds, 43592 objects whose average size is 100KB is downloaded from minio. Speed of download is coming around 70.9MB/sec which is average bandwidth utilised during this operation. Also an average of 726.4 files downloaded per second.

> Loop 1: DELETE time 18.3 secs, 933.5 deletes/sec. Slowdowns = 0
In 18.3 seconds, deletes were performed at a rate of 933.5 files per second.

# 8. Additional information:
1. For additional debugging, to log failed requests for minio
```bash
$ ./mc admin trace -v -e myminio
```

2. Root tokens generated as part of vault initialisation will not have expiry. It is always better to revoke the root token when the setup is finished. For creating a new root token when required, follow below steps

```bash
# vault operator generate-root -generate-otp
RgZjYVW2fIsl2avU5VNCpru6sZ
# vault operator generate-root -init -otp="RgZjYVW2fIsl2avU5VNCpru6sZ"
Nonce         c5cc8fb9-3226-71f2-865b-47d61bae4987
Started       true
Progress      0/3
Complete      false
OTP Length    26
# vault operator generate-root -otp="RgZjYVW2fIsl2avU5VNCpru6sZ"
Operation nonce: c5cc8fb9-3226-71f2-865b-47d61bae4987
Unseal Key (will be hidden):
Nonce       c5cc8fb9-3226-71f2-865b-47d61bae4987
Started     true
Progress    1/3
Complete    false
# vault operator generate-root -otp="RgZjYVW2fIsl2avU5VNCpru6sZ"
Operation nonce: c5cc8fb9-3226-71f2-865b-47d61bae4987
Unseal Key (will be hidden):
Nonce       c5cc8fb9-3226-71f2-865b-47d61bae4987
Started     true
Progress    2/3
Complete    false
# vault operator generate-root -otp="RgZjYVW2fIsl2avU5VNCpru6sZ"
Operation nonce: c5cc8fb9-3226-71f2-865b-47d61bae4987
Unseal Key (will be hidden):
Nonce            c5cc8fb9-3226-71f2-865b-47d61bae4987
Started          true
Progress         3/3
Complete         true
Encoded Token    IUk1JREjE1k8e0YLXDY0ZnljI3o+EDR5HGI
# vault operator generate-root -decode IUk1JREjE1k8e0YLXDY0ZnljI3o+EDR5HGI -otp RgZjYVW2fIsl2avU5VNCpru6sZ
s.oOHuDkZ25gnWB3L5m9NbAOo8
# export VAULT_TOKEN=s.oOHuDkZ25gnWB3L5m9NbAOo8
# vault token lookup
Key                 Value
---                 -----
accessor            laQoKv6pyhO3kpkA1ozmG8Ev
creation_time       1604397451
creation_ttl        0s
display_name        root
entity_id           n/a
expire_time         <nil>
explicit_max_ttl    0s
id                  s.oOHuDkZ25gnWB3L5m9NbAOo8
meta                <nil>
num_uses            0
orphan              true
path                auth/token/root
policies            [root]
ttl                 0s
type                service
3. When you need to do vault administration. Create an additional administrator app certificate as opposed to using the app certificate created for minio itself for better auditability.
# Create admin cert pair
kes tool identity new --key=minio-kes-admin.key --cert=minio-kes-admin.cert minio-kes-admin
# Generate app identity from cert
kes tool identity of minio-kes-admin.cert
# Configure required administration policies in KES server-config.yml file and reload KES service
```

3. When you need to do vault administration. Create an additional administrator app certificate as opposed to using the app certificate created for minio itself for better auditability.

```bash
# Create admin cert pair
kes tool identity new --key=minio-kes-admin.key --cert=minio-kes-admin.cert minio-kes-admin
# Generate app identity from cert
kes tool identity of minio-kes-admin.cert
# Configure required administration policies in KES server-config.yml file and reload KES service
```

4. Once a bucket is created with some content, if we enable encryption later on the bucket, existing files are not re-encrypted automatically.

5. `mc admin heal` is about to be deprecated and no sufficient alternative is in place when I checked last time.

> A note on Automatic healing:
> Healing is automatic on server side which runs on a continuous basis on a low priority thread, mc admin heal is deprecated and will be removed in future.
> - Minio healing moved from once every day to once every month
> - Option to disable self healing is under works
> - https://github.com/minio/minio/issues/7892
> - https://github.com/minio/minio/pull/7934
> - https://github.com/minio/minio/pull/7868

6. While upgrading minio in distributed mode. Minio should be brought down one after another for maintenance and the instance should be immediately healed after being up before going for other instances. This is necessary, otherwise affects the redundancy factor

- Bring down minio
- Upgrade
- Bring up minio
- Run mc admin heal to recover the missing data
- Go for next node and upgrade only if the current one’s recovery is complete

7. Minio will not tell you the health status of disks. You should have separate alerting and monitoring in place to keep track of hardware health.

8. Note that there are other reasons but servers restarts that can cause nodes to become out of sync. For example, if there is a network failure that causes some of the nodes to not be reachable, writes will be less durable too. It is thus important to have good monitoring in place and respond accordingly. Minio itself will auto-heal the cluster every month if the administrator doesn’t trigger a heal themselves.

9. `MINIO_KMS_AUTO_ENCRYPTION=on` environment variable is about to get deprecated in favour of mc encrypt command which provides encryption at rest on specific buckets. However this deprecation doesn't sound reasonable to me as we don't see any suitable alternative that provides auto encryption for all buckets on creation.

10. Unsealing vault for every restart is a pain. Fortunately, Vault exposes different integrations to make this automatic — https://learn.hashicorp.com/collections/vault/auto-unseal

## References:
Administration
- https://docs.wire.com/how-to/administrate/minio.html

Different certificate generation approaches
- https://kubernetes.io/docs/concepts/cluster-administration/certificates/
- https://gardener.cloud/documentation/guides/applications/https/
- https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/

*This article was published in medium: https://link.medium.com/iIKF9apqbfb*