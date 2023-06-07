# Vault agent inject

Exrpot enviroment host `vault`
```sh
export VAULT_ADDR='http://vault:8200'
```

Enable auth kubernets
```sh
vault auth enable kubernetes
```

Config auth in `vault`
```sh
export SERVICE_ACCOUNT_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
export PATH_TO_CERTIFICATE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt 

vault write auth/kubernetes/config \
kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
token_reviewer_jwt=$SERVICE_ACCOUNT_TOKEN \
kubernetes_ca_cert=@$PATH_TO_CERTIFICATE
```

Create service account in `k8s`
```sh
kubectl create sa k8s-service-staging --namespace staging
```

Create a policy in `vault`
```sh
vault policy write demo-app - <<EOF
path "staging/data/database/*" {
    capabilities = ["read"]
}
EOF
```

Create a role in `vault`
```sh
vault write auth/kubernetes/role/demo-app \
policies=demo-app \
bound_service_account_namespaces=staging \
bound_service_account_names=k8s-service-staging \
ttl=24h
```

Create a deployment in `k8s`
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    ...
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/agent-configmap: 'secrets-vault'
        vault.hashicorp.com/agent-init-first: 'true'
        vault.hashicorp.com/agent-pre-populate: 'true'
    ...  
```

Create a configmap in `k8s`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: secrets-vault
data:
  config-init.hcl: |
    auto_auth {
      method "kubernetes" {
        config = {
          role = "demo-app"
        }
        type = "kubernetes"
      }

      sink "file" {
        config = {
          path = "/home/vault/.token"
        }

        type = "file"
      }
    }

    exit_after_auth = true
    pid_file = "/home/vault/.pid"

    vault {
      address = "https://vault:8200"
    }
  config.hcl: |
    auto_auth {
      method "kubernetes" {
        config = {
          role = "demo-app"
        }
        type = "kubernetes"
      }

      sink "file" {
        config = {
          path = "/home/vault/.token"
        }

        type = "file"
      }
    }

    exit_after_auth = false
    pid_file = "/home/vault/.pid"

    template_config {
      exit_on_retry_failure = true
      static_secret_render_interval = "10s"
    }

    template "database-config" {
      contents = "{{- with secret \"staging/data/database/config\" -}}{{- range $k, $v := .Data.data }}{{ printf \"%s=\\\"%s\\\"\\n\" $k $v }}{{- end }}{{- end }}"
      destination = "/vault/secrets/.env-2"
    }

    vault {
      address = "https://vault:8200"
    }

```