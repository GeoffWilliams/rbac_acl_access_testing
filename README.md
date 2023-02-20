# RBAC/ACL access testing
Walkthrough of how to test confluent ACL + RBAC rule effectiveness by proof.

# Prerequisites

* RBAC enabled mTLS cluster with external access - https://github.com/GeoffWilliams/cfk_vault_mtls_rbac_walkthrough/
* Confluent platform installed and binaries in $PATH

# Setup

## Environment variables

```
# adjust as needed
export CFK_WALKTHROUGH_HOME=~/github/cfk_vault_mtls_rbac_walkthrough
```

## confluent cli

```
confluent login --url https://kafka-mds.mydomain.example:443 \
    --ca-cert-path "$CFK_WALKTHROUGH_HOME/generated/cacerts.pem" --save
username: kafka
password: kafka-secret
```

## Make test user credentials with OpenSSL

* `alice` already exists in our ldap container
* Authenticate via mTLS certificate
* **Principal is extracted from the certificate CN!**

> **Note**
> Possible to do the CSR + key generation in one step

### OpenSSL: key

> **Note**
> For funs we will use a long CN for alice, it will be extracted away by our `principalMappingRules`!

```
openssl genrsa -out generated/alice@mydomain.example.key.pem 2048
```

### OpenSSL: csr

```
openssl req -new -key generated/alice@mydomain.example.key.pem -nodes -out generated/alice@mydomain.example.csr \
    -subj "/C=AU/ST=NSW/L=McMahons Point/O=World Domination Inc./CN=alice@mydomain.example"
```

### OpenSSL: sign certificate

```
openssl x509 -req -CA $CFK_WALKTHROUGH_HOME/generated/cacerts.pem \
    -CAkey $CFK_WALKTHROUGH_HOME/generated/rootCAkey.pem \
    -in generated/alice@mydomain.example.csr \
    -out generated/alice@mydomain.example.pem \
    -days 365 -CAcreateserial
```

### Java KeyStore (JKS) for alice

#### Extract .pks12 from certificate and key
```
openssl pkcs12 -export \
	-in "generated/alice@mydomain.example.pem" \
	-inkey "generated/alice@mydomain.example.key.pem" \
	-out "generated/alice@mydomain.example.pkcs12" \
	-name "alice@mydomain.example" \
	-passout pass:mykeypassword
```

### Create keystore and import pkcs12 file

```
keytool -importkeystore \
	-deststorepass "mystorepassword" \
	-destkeypass "mystorepassword" \
	-destkeystore "generated/alice@mydomain.example.jks" \
	-deststoretype pkcs12 \
	-srckeystore "generated/alice@mydomain.example.pkcs12" \
	-srcstoretype PKCS12 \
	-srcstorepass mykeypassword
```

## Kafka connection .properties files

> **Note**
> Regenerate these to get the right path to the .jks files!

### User:kafka

```
cat <<EOF > kafka.properties
security.protocol=SSL
ssl.truststore.location=$CFK_WALKTHROUGH_HOME/generated/truststore.jks
ssl.truststore.password=mystorepassword
ssl.keystore.location=$CFK_WALKTHROUGH_HOME/generated/kafka-keystore.jks
ssl.keystore.password=mystorepassword
ssl.key.password=mystorepassword
EOF
```

### User:alice

```
cat <<EOF > alice.properties
security.protocol=SSL
ssl.truststore.location=$CFK_WALKTHROUGH_HOME/generated/truststore.jks
ssl.truststore.password=mystorepassword
ssl.keystore.location=generated/alice@mydomain.example.jks
ssl.keystore.password=mystorepassword
ssl.key.password=mystorepassword
EOF
```

## Cluster external access (test working)

```
export CFK_WALKTHROUGH_HOME=~/github/cfk_vault_mtls_rbac_walkthrough


# As principal User:kafka - should see all topics
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --list

# As principal User:alice - should see no topics
kafka-topics --command-config alice.properties --bootstrap-server kafka.mydomain.example:9092  --list
```

# Test RBAC/ACL access conditions

## No rolebinding and no ACL -> no access

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic no.rb.no.acl
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic no.rb.no.acl
```

Result: Not allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic no.rb.no.acl --from-beginning
```

Result: Not allowed

## No rolebinding and (allow) ACL -> access allowed

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic no.rb.allow.acl
```

Create ACL

```
# topic read/write
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --operation write --topic no.rb.allow.acl

# join any group
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --group '*'
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic no.rb.allow.acl
```

Result: Allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic no.rb.allow.acl --from-beginning
```

Result: Allowed

Cleanup

```
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --remove --allow-principal User:alice --operation read --group '*'
```


## No rolebinding and (deny) ACL -> no access

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic no.rb.deny.acl
```

Create ACL

```
# topic read/write
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --deny-principal User:alice --operation read --operation write --topic no.rb.deny.acl

# join any group
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --group '*'
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic no.rb.deny.acl
```

Result: Not allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic no.rb.deny.acl --from-beginning
```

Result: Not allowed

Cleanup

```
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --remove --allow-principal User:alice --operation read --group '*'
```

## Rolebinding and (allow) ACL -> access allowed

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic yes.rb.allow.acl
```

Create RBAC role binding

```
cat <<EOF | kubectl apply -f -
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-read-yes.rb.allow.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperRead
  resourcePatterns:
    - name: "yes.rb.allow.acl"
      patternType: LITERAL
      resourceType: Topic
---
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-write-yes.rb.allow.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperWrite
  resourcePatterns:
    - name: "yes.rb.allow.acl"
      patternType: LITERAL
      resourceType: Topic
EOF
```

```
# topic read/write
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --operation write --topic yes.rb.allow.acl

# join any group
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --group '*' 
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic yes.rb.allow.acl
```

Result: Allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic yes.rb.allow.acl --from-beginning
```

Result: Allowed

Cleanup

```
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --remove --allow-principal User:alice --operation read --group '*'
```

## Rolebinding and (deny) ACL -> no access

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic yes.rb.deny.acl
```

Create RBAC role binding

```
cat <<EOF | kubectl apply -f -
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-read-yes.rb.deny.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperRead
  resourcePatterns:
    - name: "yes.rb.deny.acl"
      patternType: LITERAL
      resourceType: Topic
---
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-write-yes.rb.deny.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperWrite
  resourcePatterns:
    - name: "yes.rb.deny.acl"
      patternType: LITERAL
      resourceType: Topic
EOF
```

```
# topic read/write
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --deny-principal User:alice --operation read --operation write --topic yes.rb.deny.acl

# join any group
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --group '*' 
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic yes.rb.deny.acl
```

Result: Not allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic yes.rb.deny.acl --from-beginning
```

Result: Not allowed

Cleanup

```
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --remove --allow-principal User:alice --operation read --group '*'
```

## Rolebinding and no ACL -> access allowed

Create topic as principal User:kafka

```
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic yes.rb.no.acl
```

Create RBAC role binding

```
cat <<EOF | kubectl apply -f -
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-read-yes.rb.no.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperRead
  resourcePatterns:
    - name: "yes.rb.no.acl"
      patternType: LITERAL
      resourceType: Topic
---
apiVersion: platform.confluent.io/v1beta1
kind: ConfluentRolebinding
metadata:
  name: alice-rb-write-yes.rb.no.acl
  namespace: confluent
spec:
  principal:
    type: user
    name: alice
  role: DeveloperWrite
  resourcePatterns:
    - name: "yes.rb.no.acl"
      patternType: LITERAL
      resourceType: Topic
EOF
```

```
# join any group
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --add --allow-principal User:alice --operation read --group '*' 
```

Produce

```
kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config alice.properties \
    --topic yes.rb.no.acl
```

Result: Allowed

Consume

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config alice.properties \
    --topic yes.rb.no.acl --from-beginning
```

Result: Allowed

Cleanup

```
kafka-acls --bootstrap-server kafka.mydomain.example:9092 --command-config kafka.properties \
 --remove --allow-principal User:alice --operation read --group '*'
```

# Troubleshooting

## Nothing works! not ACLs not RBACs!

Check your principal name is extracted correctly from the CN in your certificate:

```
openssl x509 -in generated/alice@mydomain.example.pem -noout -subject | sed -n '/^subject/s/^.*CN = //p'
```

Check the deployed `principalMappingRules` in CFK:
```
# Zookeeper
kubectl get zookeepers.platform.confluent.io zookeeper -o json | jq .spec.authentication.principalMappingRules

# Kafka
kubectl get kafkas.platform.confluent.io kafka -o json | jq '.spec.listeners[].authentication.principalMappingRules'
```

## List ACLs on server

```
kafka-acls --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092 --list
```

## List RBAC denied events

* How the heck do you do this? No output from this...

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config kafka.properties --topic confluent-audit-log-events | jq 'select(.data.authorizationInfo.granted == false)'
```

# Results

* Works as expected

| Condition                      | Expected       | Actual         |
| ---                            | ---            | ---            |
| No rolebinding and no ACL      | no access      | no access      |
| No rolebinding and (allow) ACL | access allowed | access allowed |
| No rolebinding and (deny) ACL  | no access      | no access      |
| Rolebinding and (allow) ACL    | access allowed | access allowed |
| Rolebinding and (deny) ACL     | no access      | no access      |
| Rolebinding and no ACL         | access allowed | no access      |