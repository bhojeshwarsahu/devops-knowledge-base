# 🔐 Vault Database Authentication for MariaDB & MongoDB

This guide details how to configure [HashiCorp Vault](https://www.vaultproject.io/) to dynamically generate database credentials for **MariaDB Galera** and **MongoDB**, and how to securely store cloud backup secrets using Vault’s **KV secrets engine**.

---

## 📘 Prerequisites

- A running instance of HashiCorp Vault (initialized & unsealed)
- Access to a MariaDB and MongoDB server (with IP, credentials)
- Admin privileges on both databases
- Vault CLI access

---

## 🔄 Part 1: Dynamic Credentials for MariaDB Galera

### 🧩 Step 1: Enable the Database Secrets Engine

```bash
vault secrets enable database
```
#### This enables the secrets engine at the default database/ path.

## 👤 Step 2: Create a User in MariaDB
Login to MariaDB and run:
```sql
CREATE USER 'vaultuser'@'%' IDENTIFIED BY 'vaultPassW0rd';

-- Option 1: Grant full access
GRANT ALL PRIVILEGES ON *.* TO 'vaultuser'@'%' WITH GRANT OPTION;

-- OR Option 2: Grant limited privileges
GRANT CREATE USER, DROP, RELOAD, PROCESS, SHOW DATABASES, SELECT, INSERT, UPDATE, DELETE, ALTER, CREATE, INDEX ON *.* TO 'vaultuser'@'%';

FLUSH PRIVILEGES;
```
## 🔌 Step 3: Configure Vault Connection to MariaDB
```bash
vault write database/config/mariadb \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(<mariadb_private_ip>:3306)/" \
    allowed_roles="mariadb-backup" \
    username="vaultuser" \
    password="vaultPassW0rd"
```
#### Replace `<mariadb_private_ip>` with the actual private IP of your MariaDB instance.
## 🎯 Step 4: Create a Vault Role for MariaDB
This defines how dynamic credentials are generated.
```bash
vault write database/roles/mariadb-backup \
  db_name=mariadb \
  creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';
GRANT SELECT, SHOW VIEW, CREATE VIEW, RELOAD, LOCK TABLES, REPLICATION CLIENT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, PROCESS ON *.* TO '{{name}}'@'%';" \
  default_ttl="1h" \
  max_ttl="24h"
```
## 🛢 Part 2: Dynamic Credentials for MongoDB
## 👤 Step 1: Create a User in MongoDB
In your MongoDB shell:
```javascript
use admin

db.createUser({
  user: "vaultuser",
  pwd: "vaultPassW0rd",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" }
  ]
})
```
## 🔌 Step 2: Configure Vault Connection to MongoDB
```bash
vault write database/config/mongodb \
    plugin_name=mongodb-database-plugin \
    allowed_roles="mongodb-backup" \
    connection_url="mongodb://{{username}}:{{password}}@<mongodb_private_ip>:27017/admin" \
    username="vaultuser" \
    password="vaultPassW0rd"
```
#### Replace `<mongodb_private_ip>` with the actual private IP of your MongoDB instance.
```bash
vault write database/roles/mongodb-backup \
  db_name=mongodb \
  creation_statements='{
    "db": "admin",
    "roles": [
      { "role": "readWriteAnyDatabase", "db": "admin" },
      { "role": "userAdminAnyDatabase", "db": "admin" },
      { "role": "dbAdminAnyDatabase", "db": "admin" },
      { "role": "read", "db": "config" },
      { "role": "read", "db": "local" }
    ]
  }' \
  default_ttl="1h" \
  max_ttl="24h"
```
## 🔐 Part 3: Storing Cloud Secrets in Vault
### 💾 Store Backup Keys & Encryption Password
```bash
vault kv put kv/backup_secrets \
  s3_access_key="access key" \
  s3_secret_key="secret key" \
  encryption_key="encryption password"

vault kv put kv/mongodb_backup \
  s3_access_key="access key" \
  s3_secret_key="secret key" \
  encryption_key="encryption password"
```
## 🛡️ Part 4: Vault Policy for AppRole Authentication
### 📄 Create a Vault Policy File
```hcl
# approle-policy.hcl

# Allow reading secrets from KV
path "kv/*" {
  capabilities = ["read", "create", "list"]
}

# Allow generating dynamic MariaDB credentials
path "database/creds/mariadb-backup" {
  capabilities = ["read"]
}

# Allow generating dynamic MongoDB credentials
path "database/creds/mongodb-backup" {
  capabilities = ["read"]
}
```
### 📝 Apply the Policy
```bash
vault policy write approle-policy approle-policy.hcl
```
## 🔄 Part 5: Generate Dynamic Credentials
#### MariaDB
```bash
vault read database/creds/mariadb-backup
```
#### MongoDB
```bash
vault read database/creds/mongodb-backup
```
#### Each call returns a new temporary username and password.

## 🗄️ Part 6: Backup Example Using Vault-Generated Credentials
### 💡 Example MongoDB Dump with Vault Credentials
```bash
mongodump \
  --uri="mongodb://<username>:<password>@127.0.0.1:27017/?authSource=admin" \
  --db=<db_name> \
  --gzip \
  --archive=/tmp/mongo_backups/<db_name>_backup_$(date +%Y%m%d_%H%M%S).gz
```
#### Replace `<username>` and `<password>` with Vault-generated credentials.

## ✅ Summary

You now have:

- 🔐 Vault securely managing dynamic credentials for **MariaDB** and **MongoDB**
- 🛡️ Role-based access control using Vault policies and AppRole
- 🔑 Cloud secrets (S3, encryption keys) stored securely using the **KV secrets engine**
- 📦 Secure, credential-rotated backups using dynamically generated DB users

> This setup improves security posture and ensures credentials are **temporary, auditable, and centrally managed**.