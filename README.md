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

## Cluster external access (test working)

```
cat <<EOF > kafka.properties
security.protocol=SSL
ssl.truststore.location=$CFK_WALKTHROUGH_HOME/generated/truststore.jks
ssl.truststore.password=mystorepassword
ssl.keystore.location=$CFK_WALKTHROUGH_HOME/generated/kafka-keystore.jks
ssl.keystore.password=mystorepassword
ssl.key.password=mystorepassword
EOF


kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --list
```

# Test RBAC/ACL access conditions

## No rolebinding and no ACL -> no access
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --create \
    --topic no_rb_no_acl

kafka-console-producer --bootstrap-server kafka.mydomain.example:9092 --producer.config kafka.properties \
    --topic no_rb_no_acl

## No rolebinding and (allow) ACL -> access allowed

## No rolebinding and (deny) ACL -> no access

## Rolebinding and (allow) ACL -> access allowed

## Rolebinding and (deny) ACL -> no access

## Rolebinding and no ACL -> access allowed


