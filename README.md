# Leafy Pay: MCK and OpenShift

## Steps:
- access the OpenShift account
- create a project (which is a namespace)
- install MCK
- deploy MCK
- deploy a replica-set

For this project, we will use the namespace `leafy-pay`


## Install MCK
To install MCK, the `Telemetry is disabled` since RBACs is required to be cluster-scoped and we don't have this access.  
Also, `runAsUser` is set to `null` to allow OpenShift to decide what user to use, since by default the Operator tries to use the user `2000` which is not allowed in OC for security reasons.

```
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update
 
helm install kubernetes-operator mongodb/mongodb-kubernetes \
   --namespace leafy-pay \
   --set operator.env=dev \
   --set operator.watchNamespace=leafy-pay \
   --set operator.telemetry.enabled=false \
   --set managedSecurityContext=true
```

## Deploy Ops Manager
Set the versions of OM and AppDB in ops-manager/deploy-om.yaml.
Then run the following command:

```
kubectl apply -f ./ops-manager/deploy-om.yaml
```

### Configure Ops Manager
Find the external IP of OM: with `k9s` go to services `:svc` and look for `ops-manager-svc-ext`.
Its IP is the one externally accessible. Access OpsManager via `http://<IP>:8080`.

Then:
- create an Organization with the name `leafy-org`
- create API Key in Access Manager (Organization Owner permissions)
- add the IP of the Operator pod to the access list
- update `config-map.yaml` and `replica-set.yaml` with the orgId, private and public keys
- update the URL of OpsManager

## Deploy replica-set
Run from the `/replica-set` folder:
```
kubectl apply -f config-map.yaml -f replica-set.yaml
```

### Building the connection string
Check the services, and there will be 3 external IPs for each member of the RS.  
Use them and build the connection string.

```
mongodb://<external-host-0>:27017,<external-host-1>:27017,<external-host-2>:27017/?replicaSet=replica-set
```

## TLS

Create the CA:
```
kubectl apply -f certificate-authority.yaml
````

Create the CA Certificate:
```
kubectl get secret leafy-root-secret \
  -n cert-manager \
  -o jsonpath="{.data['ca\\.crt']}" | base64 --decode > ca.pem
```

Create the ConfigMap:
```
kubectl create configmap "replica-set-ca-configmap" \
  -n "leafy-pay" \
  --from-file=ca-pem=./ca.pem \
  --from-file=mms-ca.crt=./ca.pem \
  --from-file=ca.crt=./ca.pem \
  --dry-run=client -o yaml | kubectl --context "k3d-mongodb-mck-cluster" apply -f -
```

Create the `leafy-root-secret`:
```
kubectl create secret generic leafy-root-secret \
  -n leafy-pay \
  --from-file=ca.crt=./ca.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

Issue the Certificates:
```
kubectl apply -f certificates.yaml
```

Deploy repli-set with TLS enabled:
```
kubectl apply -f ../replica-set/replica-set-tls.yaml
```