# RBAC/ACL access testing
Walkthrough of how to test confluent ACL + RBAC rule effectiveness by proof.

# TLDR
1. RBAC and ACL co-exist as you would expect with deny ACLs trumping RBAC bindings
2. OOTB access denied errors are only in the broker log
3. OOTB `confluent-audit-log-events` only contains platform related audit events: create topic, create rbac rule, etc
4. Enable auth logging -> `confluent-audit-log-events`: set confluent.security.event.logger.authentication.enable=true
5. Enable logging produce/consume/describe fails -> `confluent-audit-log-events` set `confluent.security.event.router.config`. The value should be a single line JSON policy (voodoo), see project readme for example.

# Proof

## Prerequisites

* RBAC enabled mTLS cluster with external access - https://github.com/GeoffWilliams/cfk_vault_mtls_rbac_walkthrough/
* Confluent platform installed and binaries in $PATH

## Setup

### Environment variables

```
# adjust as needed
export CFK_WALKTHROUGH_HOME=~/github/cfk_vault_mtls_rbac_walkthrough
```

### confluent cli

```
confluent login --url https://kafka-mds.mydomain.example:443 \
    --ca-cert-path "$CFK_WALKTHROUGH_HOME/generated/cacerts.pem" --save
username: kafka
password: kafka-secret
```

### Make test user credentials with OpenSSL

* `alice` already exists in our ldap container
* Authenticate via mTLS certificate
* **Principal is extracted from the certificate CN!**

> **Note**
> Possible to do the CSR + key generation in one step

#### OpenSSL: key

> **Note**
> For funs we will use a long CN for alice, it will be extracted away by our `principalMappingRules`!

```
openssl genrsa -out generated/alice@mydomain.example.key.pem 2048
```

#### OpenSSL: csr

```
openssl req -new -key generated/alice@mydomain.example.key.pem -nodes -out generated/alice@mydomain.example.csr \
    -subj "/C=AU/ST=NSW/L=McMahons Point/O=World Domination Inc./CN=alice@mydomain.example"
```

#### OpenSSL: sign certificate

```
openssl x509 -req -CA $CFK_WALKTHROUGH_HOME/generated/cacerts.pem \
    -CAkey $CFK_WALKTHROUGH_HOME/generated/rootCAkey.pem \
    -in generated/alice@mydomain.example.csr \
    -out generated/alice@mydomain.example.pem \
    -days 365 -CAcreateserial
```

#### Java KeyStore (JKS) for alice

##### Extract .pks12 from certificate and key
```
openssl pkcs12 -export \
	-in "generated/alice@mydomain.example.pem" \
	-inkey "generated/alice@mydomain.example.key.pem" \
	-out "generated/alice@mydomain.example.pkcs12" \
	-name "alice@mydomain.example" \
	-passout pass:mykeypassword
```

#### Create keystore and import pkcs12 file

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

### Kafka connection .properties files

> **Note**
> Regenerate these to get the right path to the .jks files!

#### User:kafka

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

#### User:alice

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

### Cluster external access (test working)

```
export CFK_WALKTHROUGH_HOME=~/github/cfk_vault_mtls_rbac_walkthrough


# As principal User:kafka - should see all topics
kafka-topics --command-config kafka.properties --bootstrap-server kafka.mydomain.example:9092  --list

# As principal User:alice - should see no topics
kafka-topics --command-config alice.properties --bootstrap-server kafka.mydomain.example:9092  --list
```

## Test RBAC/ACL access conditions

### No rolebinding and no ACL -> no access

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

### No rolebinding and (allow) ACL -> access allowed

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


### No rolebinding and (deny) ACL -> no access

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

### Rolebinding and (allow) ACL -> access allowed

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

### Rolebinding and (deny) ACL -> no access

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

### Rolebinding and no ACL -> access allowed

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

## Results

* Works as expected

| Condition                      | Expected       | Actual         |
| ---                            | ---            | ---            |
| No rolebinding and no ACL      | no access      | no access      |
| No rolebinding and (allow) ACL | access allowed | access allowed |
| No rolebinding and (deny) ACL  | no access      | no access      |
| Rolebinding and (allow) ACL    | access allowed | access allowed |
| Rolebinding and (deny) ACL     | no access      | no access      |
| Rolebinding and no ACL         | access allowed | no access      |

# Troubleshooting

## Where to find access denied errors?

On an out-of-the-box CFK/CP setup the place to look for logs is the broker log. Here are some examples:

Principal name extracted incorrectly. `User:alice.mydomain.example` should be `User:alice` to match our rules
```
kafka.authorizer.logger logAuthorization - Principal = User:alice.mydomain.example is Denied Operation = Write from host = 10.42.0.0 on resource = Topic:ANY`
```

`User:alice` denied access to topic
```
kafka.authorizer.logger logAuthorization - Principal = User:alice is Denied Operation = Describe from host = 172.21.0.1 on resource = Topic:LITERAL:rbactest
```

## What is the `confluent-audit-log-events` topic for and why doesn't it contain any RBAC/ACL logs?

Reference: https://docs.confluent.io/platform/current/security/audit-logs/audit-logs-properties-config.html#configure-audit-logs

On Confluent Platform, only these events are captured by default:

* Topics: create and delete
* ACLs: create and delete
* Authorization requests related to RBAC (I *think* this means editing the rules themselves - asked)
* Every event in the MANAGEMENT category

This means the `confluent-audit-log-events` doesn't contain allow/deny information relating to topic access!

## `confluent-audit-log-events` - enable logging of authentication

Add to kafka server settings:

```
confluent.security.event.logger.authentication.enable=true
```

This will result in `confluent-audit-log-events` like this:

```
{"specversion":"1.0","id":"fbdeefc6-2ae9-4ae9-88cc-159c4f360c0e","source":"crn:///kafka=EyVCDzKYSc2vXhyD-z9PXg","type":"io.confluent.kafka.server/authentication","datacontenttype":"application/json","subject":"crn:///kafka=EyVCDzKYSc2vXhyD-z9PXg","time":"2023-02-21T06:07:38.958333Z","route":"confluent-audit-log-events","data":{"serviceName":"crn:///kafka\u003dEyVCDzKYSc2vXhyD-z9PXg","methodName":"kafka.Authentication","resourceName":"crn:///kafka\u003dEyVCDzKYSc2vXhyD-z9PXg","authenticationInfo":{"principal":"User:alice","metadata":{"mechanism":"SSL","identifier":"CN\u003dalice@mydomain.example,O\u003dWorld Domination Inc.,L\u003dMcMahons Point,ST\u003dNSW,C\u003dAU"},"principalResourceId":""},"requestMetadata":{"client_address":"/172.21.0.1"},"result":{"status":"SUCCESS","message":""}}}
```

> **Warning**
> This only indicates status of **authentication**!

> **Note**
> Example CFK environment from https://github.com/GeoffWilliams/cfk_vault_mtls_rbac_walkthrough includes this setting but it is commented out for you to enable, matching the real world


## `confluent-audit-log-events` - enable logging of producer/consumer/topic access denied events

Add to kafka server settings:

```
confluent.security.event.router.config={big chunk of json}
```

The JSON format is described [here](https://docs.confluent.io/platform/current/security/audit-logs/audit-logs-properties-config.html#view-all-authorization-event-types). Note that the JSON must be all on one line! Or must have a `\` at the end of each line. Choose your poison.

Your advised to first look at the existing settings before adding your rules. On a fresh CFK deployment I have:

```
confluent audit-log config describe


{
  "destinations": {
    "topics": {
      "confluent-audit-log-events": {
        "retention_ms": 7776000000
      }
    }
  },
  "default_topics": {
    "allowed": "confluent-audit-log-events",
    "denied": "confluent-audit-log-events"
  },
  "metadata": {
    "resource_version": "L38PPqakrbU5zX1bqdjz_A",
    "modified_since": "2023-02-21T08:54:31Z"
  }
}
```


### Example router config to capture access denied errors

```
{
    "routes": {
        "crn:///kafka=*": {
            "interbroker": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "describe": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "management": {
                "allowed": "confluent-audit-log-events",
                "denied": "confluent-audit-log-events"
            }
        },
        "crn:///kafka=*/group=*": {
            "consume": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "describe": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "management": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            }
        },
        "crn:///kafka=*/topic=*": {
            "produce": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "consume": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            },
            "describe": {
                "allowed": "",
                "denied": "confluent-audit-log-events"
            }
        }
    },
    "destinations": {
        "topics": {
            "confluent-audit-log-events": {}
        }
    },
    "default_topics": {
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
    },
    "excluded_principals": [
        "User:kafka"
    ]
}
```

Pipe this through `jq -c` to get a single line of text.

This will result in `confluent-audit-log-events` like this:

```
{"specversion":"1.0","id":"91b3626d-4b1a-49e0-9921-bdf120ced00d","source":"crn:///kafka=EyVCDzKYSc2vXhyD-z9PXg","type":"io.confluent.kafka.server/authorization","datacontenttype":"application/json","subject":"crn:///kafka=EyVCDzKYSc2vXhyD-z9PXg/topic=no.rb.no.acl","time":"2023-02-21T10:26:58.886604Z","route":"confluent-audit-log-events","data":{"serviceName":"crn:///kafka\u003dEyVCDzKYSc2vXhyD-z9PXg","methodName":"kafka.Metadata","resourceName":"crn:///kafka\u003dEyVCDzKYSc2vXhyD-z9PXg/topic\u003dno.rb.no.acl","authenticationInfo":{"principal":"User:alice","principalResourceId":""},"authorizationInfo":{"granted":false,"operation":"Describe","resourceType":"Topic","resourceName":"no.rb.no.acl","patternType":"LITERAL"},"request":{"correlation_id":"53","client_id":"console-producer"},"requestMetadata":{"client_address":"/172.21.0.1"}}}
```

You can then get fancy and pipe the events through `jq` to do things like filter for access denied errors...

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config kafka.properties --topic confluent-audit-log-events | jq 'select(.data.authorizationInfo.granted == false)'
```

...or errors affecting a specific principal

```
kafka-console-consumer --bootstrap-server kafka.mydomain.example:9092 --consumer.config kafka.properties --topic confluent-audit-log-events | jq 'select(.data.authenticationInfo.principal == "User:alice")'
```

With `confluent-audit-log-events` containing interesting events, you can also plug in KSQL, Coran has written some example queries to route logs into different topics: https://github.com/coranstow/cc-Audit-Log-ksql/


> **Note**
> Example CFK environment from https://github.com/GeoffWilliams/cfk_vault_mtls_rbac_walkthrough includes this setting but it is commented out for you to enable, matching the real world

## Nothing works! not ACLs not RBACs!

Check your server is RBAC/ACL enabled:

```
authorizer.class.name=io.confluent.kafka.security.authorizer.ConfluentServerAuthorizer 
```

Check your principal name is extracted correctly from the CN in your certificate (mTLS):

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

## List all RBAC rules

No nice way to do this at the moment. See: [FF-9166](https://confluentinc.atlassian.net/browse/FF-9166)

### Quick and dirty ways

**1. confluent CLI**

# setup environment vars first
export CFK_WALKTHROUGH_HOME=~/github/cfk_vault_mtls_rbac_walkthrough
export CP_MDS_URL=https://kafka-mds.mydomain.example:443

export CLUSTER_ID=$(confluent cluster describe --url $CP_MDS_URL --ca-cert-path $CFK_WALKTHROUGH_HOME/generated/cacerts.pem | awk '/kafka-cluster/ { print $3}')
for ROLE in $(confluent iam rbac role list -o json | jq -r .[].name) ; do
  echo "[$ROLE]"
  confluent iam rbac role-binding list --kafka-cluster-id $CLUSTER_ID --role $ROLE
  echo
done

**2. MDS REST API**

> List all rolebindings you can run the following for each role: https://docs.confluent.io/platform/current/security/rbac/mds-api.html#post--security-1.0-lookup-role-roleName
> 
> Which gets a list of Users as a return for every role. For each User you can issue the following API:
> 
> https://docs.confluent.io/platform/current/security/rbac/mds-api.html#get--security-1.0-lookup-rolebindings-principal-principal

**3. Read the topic directly**

Rolebinding info is kept in `_confluent-metadata-auth`. This is a multi-use topic including rolebinding records and internal MDS status. 

> **Note**
> If inspecting with `kafka-console-consumer` be sure to print the key or the data will be incomplete.

> **Note**
> Because all the role bindings are kept in a topic, you are able to copy rolebindings to other kafka servers by copying (or syncing) the topic data 😎

Example:

```
{"_type":"RoleBinding","principal":"User:alice","role":"DeveloperWrite","scope":{"clusters":{"kafka-cluster":"EyVCDzKYSc2vXhyD-z9PXg"}}}   {"_type":"RoleBinding","resources":[{"resourceType":"Topic","name":"yes.rb.deny.acl","patternType":"LITERAL"}]}
```


The key indicates the principal and the value indicates the bind and looks like:

```
{"_type":"RoleBinding","resources":[{"resourceType":"Topic","name":"yes.rb.allow.acl","patternType":"LITERAL"},{"resourceType":"Topic","name":"yes.rb.no.acl","patternType":"LITERAL"}]}
```