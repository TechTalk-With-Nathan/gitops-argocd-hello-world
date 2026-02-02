![GitOps](https://img.shields.io/badge/GitOps-ArgoCD-blue)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-blue)
![Vault](https://img.shields.io/badge/Secrets-HashiCorp%20Vault-000000?logo=vault&logoColor=white)

# HashiCorp Vault : Initialization, Raft Join & Kubernetes Auth

This document explains how to **initialize HashiCorp Vault**, join additional Vault pods to a **Raft cluster**, and configure **Kubernetes authentication** so an application can securely read secrets from Vault.

This setup reflects **real-world Vault deployments** (not dev mode) and is suitable for platform engineering and GitOps-based environments.

---

## 📌 What This Covers

- Manual Vault initialization and unsealing
- Joining Vault pods to a Raft cluster
- Creating a Vault policy for an application
- Binding a Kubernetes ServiceAccount to Vault via Kubernetes Auth

---

## 🧱 Initialize Vault

Vault must be initialized once before it can be used.  
Initialization generates unseal keys and a root token.

```bash
kubectl -n vault exec -i vault-0 -- \
  vault operator init \
    -key-shares=3 \
    -key-threshold=2 \
    -format=json
```
**Initialization Details** 

`key-shares=3` : Vault generates 3 unseal keys
`key-threshold=2` : Any 2 keys are required to unseal Vault
`format=json` : Structured output for easier handling

## 🔓 Unseal Vault

After initialization, Vault starts in a sealed state.
Access the `shell` and Run the following command using any two different unseal keys:

```bash 
vault operator unseal
``` 
Repeat it for the number of `key-threshold` and you should see: `Sealed: false`

## 🔗 Join Vault Pods to the Raft Cluster
After the first Vault pod is initialized, additional Vault pods can join the Raft cluster.

**Join vault-1**

```bash
kubectl -n vault exec -i vault-1 -- \
  sh -c "vault operator raft join http://vault-0.vault-internal.vault.svc:8200"
```

**Join vault-2**

```bash
kubectl -n vault exec -i vault-2 -- \
  sh -c "vault operator raft join http://vault-0.vault-internal.vault.svc:8200"
```

## 🛡️ Create an Application Policy
This policy allows an application to read a specific secret path.

```bash
vault policy write app - <<EOF
path "secret/data/hello-world/my-secret" {
  capabilities = ["read"]
}
EOF

```
**Policy Scope**

* Read-only access
* Restricted to a single secret path
* Follows the principle of least privilege

## 🔑 Configure Kubernetes Authentication

Bind a Kubernetes ServiceAccount to the Vault policy

```bash
vault write auth/kubernetes/role/app \
  bound_service_account_names=app \
  bound_service_account_namespaces=default \
  policies=app \
  ttl=24h
```
**What This Enables**

* Pods using the `app` ServiceAccount in the default namespace can authenticate to Vault
* Vault issues tokens with the `app` policy attached
* Tokens are valid for 24 hours
