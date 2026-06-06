# Security

## Spark Authentication & Shared Secret

- __Spark authentication__ - verifies that only authorized processes (executors, shuffle clients, external systems) can communicate with the Driver and with each other; prevents rogue processes from injecting tasks or stealing data
- Based on a __shared secret__ (also called auth secret) - a random token generated per application; all processes in the application must present this secret to establish communication

### Shared Secret Mechanism

- Generated at `SparkContext` creation - `SecurityManager.generateSecretKey()` produces a cryptographically random base-64 string
- Stored in `SparkConf` under `spark._secret` (internal key; not user-visible)
- Distributed to executors via the cluster manager's secure channel (YARN token, K8s secret, Standalone RPC)
- Every Netty connection (RPC, shuffle, block transfer) performs a challenge-response handshake using HMAC-SHA256 over the shared secret before any data flows

### Enabling Authentication

```python
spark = SparkSession.builder \
    .config("spark.authenticate", "true") \           # enable auth (default false)
    .config("spark.authenticate.secret", "my-secret") \  # or auto-generated
    .getOrCreate()
```

- On YARN - secret automatically generated and distributed via YARN tokens; `spark.authenticate.secret` not needed
- On Kubernetes - secret stored as K8s Secret object; mounted into executor pods
- On Standalone - shared secret must be pre-configured or passed via `spark.authenticate.secret`

### Authentication Handshake

- Netty `AuthRpcHandler` wraps all RPC channels when auth enabled -
    1. Client sends `ClientAuth(appId, challenge)` where `challenge` is a random nonce
    2. Server computes `HMAC-SHA256(secret, challenge)` and sends back
    3. Client verifies HMAC; if correct, connection established
    4. All subsequent messages on the channel are authenticated
- Shuffle service authentication - same HMAC handshake between executor and `ExternalShuffleService`

### Security Manager

- `SecurityManager` - the central auth/ACL component; lives in `SparkEnv`
- Responsibilities -
    - Holds the shared secret
    - Checks ACLs for Spark UI access, job/stage modification
    - Manages SSL context for encrypted connections
    - Integrates with Hadoop `UserGroupInformation` for Kerberos

---

## Encryption in Transit (SSL/TLS)

- __Encryption in transit__ - encrypts all data moving between Spark components (Driver↔Executor RPC, shuffle data, block transfers, Spark UI) to prevent network eavesdropping
- Spark uses TLS (via Java's `SSLContext` and Netty's SSL handler) for all network communication when enabled

### RPC Encryption

```python
spark = SparkSession.builder \
    .config("spark.network.crypto.enabled", "true") \     # AES encryption for RPC (faster than SSL)
    .config("spark.network.crypto.keyLength", "128") \    # AES key length: 128 or 256 bits
    .config("spark.network.crypto.keyFactoryAlgorithm", "PBKDF2WithHmacSHA1") \
    .getOrCreate()
```

- `spark.network.crypto.enabled=true` - enables AES-CTR encryption on all Netty channels; faster than SSL/TLS for bulk data transfer
- Key derived from the shared authentication secret via PBKDF2
- Encrypts all shuffle data, RPC messages, and block transfers

### SSL/TLS Configuration

```python
spark = SparkSession.builder \
    .config("spark.ssl.enabled", "true") \
    .config("spark.ssl.protocol", "TLSv1.3") \
    .config("spark.ssl.keyStore", "/path/to/keystore.jks") \
    .config("spark.ssl.keyStorePassword", "keystore-password") \
    .config("spark.ssl.trustStore", "/path/to/truststore.jks") \
    .config("spark.ssl.trustStorePassword", "truststore-password") \
    .config("spark.ssl.needClientAuth", "true") \          # mutual TLS
    .getOrCreate()
```

### Component-Specific SSL

- SSL configured per-component (each has its own namespace) -
    - `spark.ssl.ui.*` - Spark UI HTTPS
    - `spark.ssl.standalone.*` - Standalone cluster manager
    - `spark.ssl.historyServer.*` - History Server HTTPS
    - `spark.ssl.*` (global) - applies to all components unless overridden

### Spark UI HTTPS

```python
spark = SparkSession.builder \
    .config("spark.ui.enabled", "true") \
    .config("spark.ssl.ui.enabled", "true") \
    .config("spark.ssl.ui.port", "4440") \
    .config("spark.ssl.ui.keyStore", "/path/to/ui-keystore.jks") \
    .config("spark.ssl.ui.keyStorePassword", "password") \
    .config("spark.ssl.ui.protocol", "TLSv1.3") \
    .getOrCreate()
```

### Network Crypto vs SSL Performance

| Aspect | AES Network Crypto | SSL/TLS |
| --- | --- | --- |
| Overhead | $5-15\%$ CPU | $15-30\%$ CPU |
| Latency | Lower | Higher (handshake) |
| Key exchange | Derived from shared secret | PKI certificates |
| Use case | Shuffle data encryption | UI, History Server |
| Config | `spark.network.crypto.enabled` | `spark.ssl.enabled` |

---

## Encryption at Rest

- __Encryption at rest__ - encrypts data stored on disk (shuffle files, spill files, cached RDD data, event logs, checkpoint data)
- Spark itself does not implement at-rest encryption for most data; relies on underlying storage systems and OS features

### Shuffle and Spill File Encryption

- `spark.local.dir` - the local disk directory for shuffle files and spill files
- Native Spark encryption for local files -
```python
    spark.conf.set("spark.io.encryption.enabled", "true")     # encrypt shuffle and spill files
    spark.conf.set("spark.io.encryption.keySizeBits", "128")  # AES key size: 128 or 256
    spark.conf.set("spark.io.encryption.keygen.algorithm", "HmacSHA1")
```
- Encryption key derived from shared authentication secret; different key per application
- Encrypted at write time; decrypted at read time; transparent to application

### HDFS Encryption

- HDFS transparent encryption zones - all data written to an HDFS path in an encryption zone is automatically encrypted at the DataNode level
- Spark sees plaintext; HDFS handles encryption transparently using HDFS KMS (Key Management Server)
- Configure encryption zones in HDFS; point Spark checkpoint/eventLog dirs to encrypted zone
- No Spark-side configuration needed; encryption handled by HDFS `EncryptionZone`

### S3 Encryption

```python
# Server-side encryption (S3 manages keys)
spark = SparkSession.builder \
    .config("spark.hadoop.fs.s3a.server-side-encryption-algorithm", "AES256") \
    .getOrCreate()

# SSE-KMS (AWS KMS manages keys)
spark = SparkSession.builder \
    .config("spark.hadoop.fs.s3a.server-side-encryption-algorithm", "SSE-KMS") \
    .config("spark.hadoop.fs.s3a.server-side-encryption.key",
            "arn:aws:kms:region:account:key/key-id") \
    .getOrCreate()

# Client-side encryption (Spark encrypts before upload)
# Handled by AWS SDK CSE-KMS; configure via s3a CSE settings
```

### Event Log Encryption

```python
spark = SparkSession.builder \
    .config("spark.eventLog.enabled", "true") \
    .config("spark.eventLog.dir", "hdfs://encrypted-zone/spark-logs") \  # HDFS encryption zone
    .getOrCreate()
```

---

## Kerberos Integration

- __Kerberos__ - the network authentication protocol used in enterprise Hadoop clusters; Spark must obtain Kerberos credentials (tickets) to access HDFS, Hive Metastore, HBase, Kafka, and other Kerberized services

### Kerberos Basics in Spark Context

- __Principal__ - the Kerberos identity (`user@REALM` or `spark/hostname@REALM`)
- __Keytab__ - file containing the principal's credentials; allows automatic ticket renewal without user interaction
- __TGT (Ticket Granting Ticket)__ - the primary Kerberos credential; obtained by authenticating with KDC using keytab
- __Service ticket__ - obtained from TGT to authenticate to a specific service (HDFS NameNode, HMS, etc.)

### Kerberos Authentication in Spark

```python
# spark-submit with Kerberos
# spark-submit \
#   --principal spark-user@MYREALM.COM \
#   --keytab /path/to/spark.keytab \
#   my_job.py

spark = SparkSession.builder \
    .config("spark.kerberos.principal", "spark-user@MYREALM.COM") \
    .config("spark.kerberos.keytab", "/path/to/spark.keytab") \
    .getOrCreate()
```

### Hadoop Token Distribution

- Spark cannot pass Kerberos TGTs to executors (TGTs are not safely distributable)
- Instead uses __Hadoop delegation tokens__ -
    1. Driver authenticates with Kerberos using keytab
    2. Driver obtains delegation tokens from each service (HDFS, HMS, HBase)
    3. Delegation tokens serialized and included in YARN `ApplicationSubmissionContext` (YARN) or K8s Secret (K8s)
    4. Executors use delegation tokens to access services; no Kerberos principal needed on executors
    5. Tokens have limited lifetime; Driver renews them periodically

### Token Renewal

- Delegation tokens expire (default $24h$ for HDFS)
- Long-running jobs (Structured Streaming, weeks-long batch) require token renewal
- `spark.security.credentials.hdfs.enabled=true` (default `true`) - enables automatic HDFS token renewal
- `spark.security.credentials.hive.enabled=true` - enables HMS token renewal
- Driver's `CredentialManager` renews tokens in background before expiry; redistributes to executors via shuffle service credential update API

### Credential Providers

- Each service has a `CredentialProvider` implementation -
    - `HadoopFSCredentialProvider` - HDFS, WASB, GCS tokens
    - `HiveCredentialProvider` - Hive Metastore tokens
    - `HBaseCredentialProvider` - HBase tokens
- Custom credential providers for custom Kerberized services -
```scala
    class MyServiceCredentialProvider extends CredentialProvider {
        override val serviceName = "my-service"
        override def obtainCredentials(conf, sparkConf, hadoopConf): Option[(Credentials, Long)] = {
            val token = MyService.getDelegationToken(principal, keytab)
            val creds = new Credentials()
            creds.addToken(new Text("my-service-token"), token)
            Some((creds, token.getExpiry))
        }
    }
    // Register via ServiceLoader in META-INF/services
```

---

## ACLs & UI Security

- __ACLs (Access Control Lists)__ - control which users can view or modify running Spark applications via the Spark UI and REST API

### UI Authentication

```python
spark = SparkSession.builder \
    .config("spark.ui.enabled", "true") \
    .config("spark.acls.enable", "true") \            # enable ACL enforcement
    .config("spark.admin.acls", "spark-admin,ops") \  # users with full admin access
    .config("spark.ui.view.acls", "alice,bob") \      # users who can VIEW the UI
    .config("spark.modify.acls", "alice") \           # users who can modify jobs (cancel, etc.)
    .getOrCreate()
```

### ACL Wildcards and Groups

- `*` - grants access to all users (use only for development)
- `spark.ui.view.acls.groups=spark-users` - all members of `spark-users` Unix group can view UI
- `spark.modify.acls.groups=spark-admins` - group-based modify ACLs
- Groups resolved via configured `GroupMappingServiceProvider` (Unix groups, LDAP, etc.)

### Filter-Based UI Authentication

- For production UI security - use HTTP filter authentication
- `spark.ui.filters=org.apache.hadoop.security.authentication.server.AuthenticationFilter` - Kerberos SPNEGO authentication for Spark UI
- Requires Hadoop auth filter configuration pointing to KDC

### History Server ACLs

```python
# history-server config
spark.history.ui.acls.enable=true
spark.history.ui.admin.acls=history-admin
spark.history.ui.admin.acls.groups=history-admins
```

- By default History Server shows all apps to all users
- With ACLs enabled - users can only view their own applications; admins see all

### Job and Stage Cancellation ACLs

- `spark.modify.acls` - users listed can cancel jobs, stages, and tasks via UI or REST API
- Without proper ACLs - any user with UI access can cancel running jobs (denial of service)

---

## Spark on YARN with Kerberos

- YARN with Kerberos is the most complex Spark security configuration; requires coordination between Spark, YARN, HDFS, and the KDC

### Full Stack Configuration

```bash
# spark-submit for Kerberos-secured YARN cluster
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --principal spark-svc@COMPANY.COM \
  --keytab /etc/security/keytabs/spark.keytab \
  --conf spark.authenticate=true \
  --conf spark.network.crypto.enabled=true \
  --conf spark.io.encryption.enabled=true \
  --conf spark.ssl.enabled=true \
  --conf spark.yarn.historyServer.address=https://history-server:18480 \
  my_job.py
```

### YARN Token Flow

1. `spark-submit` kinit using `--keytab` and `--principal`
2. `YarnClientSchedulerBackend` submits application to YARN RM -
    - Obtains HDFS delegation token from NameNode
    - Obtains HMS delegation token (if Hive configured)
    - Bundles tokens into `ContainerLaunchContext` for AM
3. YARN RM launches Application Master (AM) container with bundled tokens
4. AM (Driver in cluster mode) obtains additional tokens if needed
5. AM passes tokens to executor container launch contexts via YARN
6. Executors start with tokens; access HDFS/HMS using delegation tokens
7. `CredentialManager` in AM renews tokens before expiry; pushes renewed tokens to executors via `YarnSchedulerBackend`

### Keytab in Cluster Mode

- In cluster mode, Driver runs on a cluster node → needs keytab accessible on that node
- Options -
    - `--keytab` with HDFS path - Spark copies keytab to HDFS; distributes to AM container
    - Pre-provisioned keytab on all cluster nodes (via Ansible/Chef)
    - HashiCorp Vault or Kubernetes Secrets (for K8s; see next section)

### YARN Queue and Permissions

```python
spark = SparkSession.builder \
    .config("spark.yarn.queue", "production-queue") \   # submit to specific YARN queue
    .config("spark.yarn.maxAppAttempts", "4") \         # retries for AM failure
    .getOrCreate()
```

- YARN queue ACLs - configured in `capacity-scheduler.xml`; control which users/groups can submit to which queues
- Spark jobs submitted to unauthorized queues → `AccessControlException` at submission time

---

## Secrets Management on Kubernetes

- Kubernetes provides native secrets management for Spark; avoids embedding credentials in container images or Spark configs

### Kubernetes Secrets for Spark Auth

- Spark on K8s automatically creates and manages the shared authentication secret as a K8s Secret -
    - Driver pod creates a K8s Secret with a random auth token
    - Executor pods mount the secret as an environment variable
    - All auth handshakes use this token
    - Secret deleted when application completes

### Custom Secrets via K8s Secrets API

```python
# Reference K8s secret in Spark config
spark = SparkSession.builder \
    .config("spark.kubernetes.driver.secretKeyRef.MY_DB_PASSWORD",
            "my-db-secret:password") \         # K8s Secret name:key
    .config("spark.kubernetes.executor.secretKeyRef.MY_DB_PASSWORD",
            "my-db-secret:password") \
    .getOrCreate()

# Inside task code:
import os
db_password = os.environ["MY_DB_PASSWORD"]
```

### Mounting Secrets as Files

```python
# Mount entire K8s Secret as directory
spark = SparkSession.builder \
    .config("spark.kubernetes.driver.secrets.my-tls-secret", "/etc/tls") \
    .config("spark.kubernetes.executor.secrets.my-tls-secret", "/etc/tls") \
    .getOrCreate()

# Files: /etc/tls/tls.crt, /etc/tls/tls.key available in container
```

### Keytab on Kubernetes

```python
# Store keytab as K8s Secret
# kubectl create secret generic spark-keytab --from-file=spark.keytab=/local/path/spark.keytab

spark = SparkSession.builder \
    .config("spark.kerberos.principal", "spark@REALM.COM") \
    .config("spark.kerberos.keytab", "/mnt/secrets/spark.keytab") \
    .config("spark.kubernetes.driver.secrets.spark-keytab", "/mnt/secrets") \
    .config("spark.kubernetes.executor.secrets.spark-keytab", "/mnt/secrets") \
    .getOrCreate()
```

### Service Account RBAC

```yaml
# K8s RBAC for Spark service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark
  namespace: spark-jobs
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spark-role
  namespace: spark-jobs
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["create", "get", "list", "watch", "delete", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: spark-jobs
subjects:
- kind: ServiceAccount
  name: spark
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
```

```python
# Use service account
spark = SparkSession.builder \
    .config("spark.kubernetes.authenticate.driver.serviceAccountName", "spark") \
    .getOrCreate()
```

### Vault Integration

- HashiCorp Vault provides dynamic secrets and fine-grained access control
- Vault Agent Injector pattern for Kubernetes -
    - Vault Agent runs as sidecar in Spark pods
    - Agent authenticates to Vault using K8s service account token
    - Agent fetches secrets and writes to shared volume
    - Spark task reads secrets from file (not env var; more secure)

```yaml
# Pod annotation to trigger Vault Agent injection
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "spark-job-role"
  vault.hashicorp.com/agent-inject-secret-db-password: "secret/data/spark/db"
  vault.hashicorp.com/agent-inject-template-db-password: |
    {{- with secret "secret/data/spark/db" -}}
    {{ .Data.data.password }}
    {{- end }}
```

### Image Pull Secrets

```python
# Private container registry credentials
spark = SparkSession.builder \
    .config("spark.kubernetes.container.image",
            "my-registry.example.com/spark:3.5.0") \
    .config("spark.kubernetes.container.image.pullSecrets",
            "my-registry-secret") \   # K8s imagePullSecrets name
    .getOrCreate()
```

### Network Policies for Spark Pods

```yaml
# Restrict which pods can communicate with Spark executor pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: spark-network-policy
  namespace: spark-jobs
spec:
  podSelector:
    matchLabels:
      spark-role: executor
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          spark-role: driver     # only driver can reach executors
    ports:
    - port: 7078               # BlockManager port
    - port: 7079               # executor port
  egress:
  - to:
    - podSelector:
        matchLabels:
          spark-role: driver
```

> [!NOTE]
> Security configuration priority for production Spark -
> 1. __Authentication always__ - `spark.authenticate=true`; prevents unauthorized executor attachment
> 2. __Network encryption__ - `spark.network.crypto.enabled=true` for shuffle/RPC; SSL for UI
> 3. __At-rest encryption__ - `spark.io.encryption.enabled=true` for shuffle/spill files + HDFS/S3 encryption zones for persistent data
> 4. __Kerberos for enterprise__ - mandatory for Kerberized YARN/HDFS clusters; keytab + delegation token lifecycle managed carefully
> 5. __K8s secrets for cloud__ - never embed credentials in images or Spark configs; use K8s Secrets with RBAC; Vault for dynamic secrets

> [!TIP]
> Security debugging checklist -
> ```
> Authentication failure:
> - "Error: Could not find or load main class" with 401 → auth secret mismatch
> - Check spark.authenticate=true on both driver and executor
> - On YARN: ensure delegation tokens generated and distributed
>
> Kerberos failure:
> - "GSS initiate failed" → TGT expired or keytab path wrong
> - "Server not found in Kerberos database" → service principal misconfigured
> - Check: klist -k /path/to/spark.keytab → list principals in keytab
> - Check: kinit -kt /path/to/spark.keytab spark-user@REALM && klist
>
> K8s secrets failure:
> - "secret not found" → secret name or namespace mismatch
> - "forbidden" → service account missing RBAC permission for secrets
> - Check: kubectl get secret my-secret -n spark-jobs
> - Check: kubectl auth can-i get secrets --as=system:serviceaccount:spark-jobs:spark
> ```