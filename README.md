# openshift-datahub

Notes for getting the OpenSource Data Catalog [datahub](https://datahubproject.io/) running on OpenShift.

Quick Install
- https://github.com/acryldata/datahub-helm

```bash
helm repo add datahub https://helm.datahubproject.io/

oc new-project datahub

# Privilege required
oc adm policy add-scc-to-user privileged system:serviceaccount:$(oc project -q):default
oc adm policy add-scc-to-user privileged system:serviceaccount:$(oc project -q):prerequisites-kafka
oc adm policy add-scc-to-user privileged system:serviceaccount:$(oc project -q):prerequisites-mysql

# install pre-reqs, scale Elastic to one for now. Set pwd (else charts look for pre-existing secrets)
helm install prerequisites datahub/datahub-prerequisites --set elasticsearch.replicas=1 --namespace datahub
helm upgrade --install prerequisites datahub/datahub-prerequisites \
  --set elasticsearch.replicas=1 \
  --set mysql.auth.existingSecret="" \
  --set mysql.auth.rootPassword=password \
  --set mysql.auth.password=password \
  --set neo4j-community.enabled=true \
  --set neo4j-community.existingPasswordSecret="" \
  --set neo4j-community.neo4jPassword=password \
  --version=0.0.14 \
  --namespace datahub

# install datahub
helm upgrade --install datahub datahub/datahub \
  --set mysqlSetupJob.password.value=password \
  --set global.sql.password.value=password \
  --set global.sql.datasource.password.value=password \
  --set global.neo4j.password.value=password \
  --version=datahub-0.2.151 \
  --namespace datahub \
  --debug

# set tmp volume for now, could probs be done via Chart
oc set volume deployment/datahub-acryl-datahub-actions --add --overwrite -t emptyDir --name=tmp --mount-path=/tmp/datahub

# needed for ingestion else tries to write to root folder
oc set env deployment/datahub-acryl-datahub-actions DATAHUB_CONFIG_PATH=/tmp/datahub
oc set env deployment/datahub-acryl-datahub-actions HOME=/tmp/datahub

# setup logs dir for ingest after volume mount above
oc exec datahub-acryl-datahub-actions-567d5c744f-lqrn8 -- mkdir -p /tmp/datahub/logs/

# expose Route for now
oc expose svc datahub-datahub-frontend --port=9002
oc patch route/datahub-datahub-frontend --type=json -p '[{"op":"add", "path":"/spec/tls", "value":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}]'

# login to Route as datahub/datahub - no OAuth configured yet
```

Trino Hive Catalog entry ingest example.

```bash
# TLS - For ingesting from self signed trino instance .. copy ca from trino into container
# could tidy this up with a Dockerfile ?
oc cp ~/git/rainforest/supply-chain/trino/trino-certs/ca.crt datahub-acryl-datahub-actions-8666b4d5b5-w2k2t:/tmp/ca.crt
oc exec datahub-acryl-datahub-actions-8666b4d5b5-w2k2t -- sh -c "openssl x509 -in /tmp/ca.crt -text >> /tmp/datahub/ingest/venv-trino-0.10.0/lib/python3.10/site-packages/certifi/cacert.pem"

#
# Ingest hive table from trino -> s3 csv
#
# select * from hive.default.wine_quality;
#
# In Ingest UI
#
# Creds: ldap creds for user1
# Database: hive
# Schemas: default
# Advanced: Enable Column Profiling
```
