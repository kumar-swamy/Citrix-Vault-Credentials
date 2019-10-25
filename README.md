Generally tier 1 ADC is being controller by network administrators but kubernetes may be managed by Developers. CIC require ADC username and password to be specified for it to configure ADC. Specifying the ADC credentials as part of CIC spec may lead to credentials leak, thus not very secure. In this regard, following documentation shows how to store the credentials in Vault server and use the Vault init container and console template container to pass the ADD credentials to CIC during CIC startup.

Note: This tutorial assumes you've setup a Vault server and enabled KV secret store.

## Step 1: Create a service account For kubernetes Auth
Log in to Kubernetes cluster and perform the following.

A. Create a service account 'cic-k8s-role' in Kubernetes

   kubectl create serviceaccount cic-k8s-role

B. Give the 'cic-k8s-role' service account permissions to create tokenreviews.authentication.k8s.io at the cluster scope.
Review the given [cic-k8s-role-service-account.yaml](manifests/cic-k8s-role-service-account.yaml)


```
$ cat cic-k8s-role-service-account.yml

...
...

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: cic-k8s-role
  namespace: default
...
...

```
Apply `cic-k8s-role-service-account.yaml`

```
$ kubectl apply -f cic-k8s-role-service-account.yaml
serviceaccount/cic-k8s-role created
clusterrole.rbac.authorization.k8s.io/cic-k8s-role configured
clusterrolebinding.rbac.authorization.k8s.io/cic-k8s-role configured
clusterrolebinding.rbac.authorization.k8s.io/role-tokenreview-binding configured
```

C. Get the token for the cic-k8s-role service account

Set VAULT_SA_NAME to the service account you created earlier
```
$ export VAULT_SA_NAME=$(kubectl get sa cic-k8s-role -o jsonpath="{.secrets[*]['name']}")
```

Set SA_JWT_TOKEN value to the service account JWT used to access the TokenReview API
```
$ export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
```


D. Get Kubernetes CA Cert used to talk to Kubernetes API

```
export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
```


## Step2: Create KV secret and setup Kubernetes Auth

Login to Vault and perform the following

A. Create a read-only policy, `citrix-adc-cred-ro`  in Vault.

 Create a policy file, [citrix-adc-kv-ro.hcl](manifests/citrix-adc-kv-ro.hcl)
```
 $ tee citrix-adc-kv-ro.hcl <<EOF
 # If working with K/V v1
 path "secret/citrix-adc/*" {
     capabilities = ["read", "list"]
 }

 # If working with K/V v2
 path "secret/data/citrix-adc/*" {
     capabilities = ["read", "list"]
 }
 EOF

 # Create a policy named citrix-adc-kv-ro
 $ vault policy write citrix-adc-kv-ro citrix-adc-kv-ro.hcl

```

B. Create a KV secret with Citrix ADC credentials at the secret/citrix-adc/ path

```
vault kv put secret/citrix-adc/credential username='nsroot' \
        password='nsroot' \
        ttl='30m'
```

C. Setup the Kubernetes auth backend

Enable the Kubernetes auth method at the default path ("auth/kubernetes")
```
$ vault auth enable kubernetes
```

D. Tell Vault how to communicate with the Kubernetes  cluster
```
$ vault write auth/kubernetes/config \
        token_reviewer_jwt="$SA_JWT_TOKEN" \
        kubernetes_host="https://<K8S_CLUSTER_URL>:<API_SERVER_PORT>" \
        kubernetes_ca_cert="$SA_CA_CRT"
```

E. Authorization with this backend is role based. Before a token can be used to login, it must be configured in a role.Create a role named, 'cic-vault-example' to map Kubernetes Service Account to  Vault policies and default token TTL.

```
$ vault write auth/kubernetes/role/cic-vault-example\
        bound_service_account_names=cic-k8s-role \
        bound_service_account_namespaces=default \
        policies=citrix-adc-kv-ro \
        ttl=24h
```

This role authorizes the "cic-k8s-role" service account in the default namespace and it gives it the citrix-adc-kv-ro policy.


## Step3: Leverage Vault Agent Auto-Auth for CIC using Vault init container and consul template

![Alt text](images/cic-vault-consul-template.png?raw=true "CIC with Vault and consul-template as init containers")


A. Review the provided Vault Agent configuration file, [vault-agent-config.hcl](manifests/vault-agent-config.hcl)

```
$ cat vault-agent-config.hcl
exit_after_auth = true
pid_file = "/home/vault/pidfile"

auto_auth {
    method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
            role = "cic-vault-example"
        }
    }

    sink "file" {
        config = {
            path = "/home/vault/.vault-token"
        }
    }
}
```


Notice that the Vault Agent Auto-Auth is configured to use the kubernetes auth method enabled at the auth/kubernetes path on the Vault server. The Vault Agent will use the cic-vault-example role to authenticate.

The sink block specifies the location on disk where to write tokens. Vault Agent Auto-Auth sink can be configured multiple times if you want Vault Agent to place the token into multiple locations. In this example, the sink is set to /home/vault/.vault-token.


B. Review the provided Consul template file, [consul-template-config.hcl](manifests/consul-template-config.hcl)

```
$ cat consul-template-config.hcl

vault {
  renew_token = false
  vault_agent_token_file = "/home/vault/.vault-token"
  retry {
    backoff = "1s"
  }
}

template {
  destination = "/etc/citrix/.env"
  contents = <<EOH
  {{- with secret "secret/citrix-adc/credential" }}
  NS_USER={{ .Data.data.username }}
  NS_PASSWORD={{ .Data.data.password }}
  {{ end }}
  EOH
}
```
This template reads secrets at the secret/citrix-adc/credential path and set the username and password values.

Note: If you are using KV store version 1, use following template

```

template {
      destination = "/etc/citrix/.env"
      contents = <<EOH
      {{- with secret "secret/citrix-adc/credential" }}
      NS_USER={{ .Data.username }}
      NS_PASSWORD={{ .Data.password }}
      {{ end }}
      EOH
    }
```


C. Create a kubernetes  config-map from vault-agent-config.hcl and consul-template-config.hcl.

```

kubectl create configmap example-vault-agent-config --from-file=./vault-agent-config.hcl --form-file=./consul-template-config.hcl

```

D. Review the provided [citrix-k8s-ingress-controller-vault.yaml](manifests/citrix-k8s-ingress-controller-vault.yaml) which is a Pod specification for Citrix Ingress Controller with Vault and Consul-template as init containers.
Modify the template yaml.  When applied, Vault will fetch the token using kubernetes auth and pass it on to consul template which will create the .env file on shared volume and this will be used by CIC for authentication with tier-1 ADC.

```
$ cat citrix-k8s-ingress-controller-vault.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
  name: cic-vault
  namespace: default
spec:
  containers:
  - args:
    - --ingress-classes tier-1-vpx
    - --feature-node-watch true
    env:
    - name: NS_IP
      value: <Tier 1 ADC IP-ADDRESS>
    - name: EULA
      value: "yes"
    image: in-docker-reg.eng.citrite.net/cpx-dev/kumar-cic:latest
    imagePullPolicy: Always
    name: cic-k8s-ingress-controller
    volumeMounts:
    - mountPath: /etc/citrix
      name: shared-data
  initContainers:
  - args:
    - agent
    - -config=/etc/vault/vault-agent-config.hcl
    - -log-level=debug
    env:
    - name: VAULT_ADDR
      value: <VAULT URL>
    image: vault
    imagePullPolicy: Always
    name: vault-agent-auth
    volumeMounts:
    - mountPath: /etc/vault
      name: config
    - mountPath: /home/vault
      name: vault-token
  - args:
    - -config=/etc/consul-template/consul-template-config.hcl
    - -log-level=debug
    - -once
    env:
    - name: HOME
      value: /home/vault
    - name: VAULT_ADDR
      value: <VAULT_URL>
    image: hashicorp/consul-template:alpine
    imagePullPolicy: Always
    name: consul-template
    volumeMounts:
    - mountPath: /home/vault
      name: vault-token
    - mountPath: /etc/consul-template
      name: config
    - mountPath: /etc/citrix
      name: shared-data
  serviceAccountName: vault-auth
  volumes:
  - emptyDir:
      medium: Memory
    name: vault-token
  - configMap:
      defaultMode: 420
      items:
      - key: vault-agent-config.hcl
        path: vault-agent-config.hcl
      - key: consul-template-config.hcl
 name: example-vault-agent-config
    name: config
  - emptyDir:
      medium: Memory
    name: shared-data
```


Apply this YAML.

```
kubectl apply -f cic-vault.yaml
```
If everything goes well, Vault will fetch a token and pass it on to Consul template container. Consul template uses the token to read the ADC credentials and write it as environment variable in the path /etc/citrix/.env
CIC uses the credentials for communicating with Tier-1 ADC.

Verify that CIC is running succesfully using the credentials fetched from Vault.
