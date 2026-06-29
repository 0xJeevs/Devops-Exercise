# Exercise 20 – External Secrets Integration

## Objective

Integrate AWS Secrets Manager with Kubernetes using the External Secrets Operator.

The goal is to securely store sensitive application credentials in AWS Secrets Manager and automatically synchronize them as Kubernetes Secrets.

---

## Architecture

```
                +----------------------+
                | AWS Secrets Manager  |
                +----------+-----------+
                           |
                           |
                    External Secrets
                         Operator
                           |
                           |
                +----------v-----------+
                |   ExternalSecret     |
                +----------+-----------+
                           |
                           |
                +----------v-----------+
                | Kubernetes Secret    |
                +----------+-----------+
                           |
                           |
                    Application Pod
```

---

## Requirements

Store the following secrets in **AWS Secrets Manager**:

- DB_USERNAME
- DB_PASSWORD
- JWT_SECRET

Automatically create a Kubernetes Secret containing these values.

---

## Prerequisites

- Kubernetes Cluster
- AWS CLI configured
- IAM Role / IAM User with access to AWS Secrets Manager
- External Secrets Operator installed
- kubectl configured
- AWS Secret Store configured

---

## Step 1 – Create Secret in AWS Secrets Manager

Create a secret named:

```
app-secrets
```

Example secret value:

```json
{
  "DB_USERNAME": "admin",
  "DB_PASSWORD": "Password@123",
  "JWT_SECRET": "my-super-secret-jwt-key"
}
```

---

## Step 2 – Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm repo update

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

Verify installation:

```bash
kubectl get pods -n external-secrets
```

---

## Step 3 – Create SecretStore

Example:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secret-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-secret
            key: access-key
          secretAccessKeySecretRef:
            name: aws-secret
            key: secret-access-key
```

Apply:

```bash
kubectl apply -f secretstore.yaml
```

---

## Step 4 – Create ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secret
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore

  target:
    name: app-secret

  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: app-secrets
        property: DB_USERNAME

    - secretKey: DB_PASSWORD
      remoteRef:
        key: app-secrets
        property: DB_PASSWORD

    - secretKey: JWT_SECRET
      remoteRef:
        key: app-secrets
        property: JWT_SECRET
```

Apply:

```bash
kubectl apply -f externalsecret.yaml
```

---

## Step 5 – Verify Resources

Check the External Secret:

```bash
kubectl get externalsecret
```

Expected output:

```
NAME          STORETYPE      STORE              READY
app-secret    SecretStore    aws-secret-store   True
```

---

Check the generated Kubernetes Secret:

```bash
kubectl get secret
```

Expected output:

```
NAME         TYPE     DATA
app-secret   Opaque   3
```

Describe the Secret:

```bash
kubectl describe secret app-secret
```

---

## Decode Secret Values

Retrieve the stored values:

```bash
kubectl get secret app-secret \
-o jsonpath="{.data.DB_USERNAME}" | base64 -d
```

```bash
kubectl get secret app-secret \
-o jsonpath="{.data.DB_PASSWORD}" | base64 -d
```

```bash
kubectl get secret app-secret \
-o jsonpath="{.data.JWT_SECRET}" | base64 -d
```

---

## Project Structure

```
.
├── README.md
├── secretstore.yaml
├── externalsecret.yaml
└── screenshots
    ├── externalsecret.png
    ├── kubernetes-secret.png
    └── decoded-secret.png
```

---

## Validation

Verify the following:

```bash
kubectl get externalsecret
```

```bash
kubectl get secret
```

```bash
kubectl describe externalsecret app-secret
```

```bash
kubectl describe secret app-secret
```

All resources should be created successfully, and the Kubernetes Secret should automatically synchronize with AWS Secrets Manager.

---

## Outcome

Successfully integrated AWS Secrets Manager with Kubernetes using the External Secrets Operator. Sensitive credentials are securely managed in AWS and automatically synchronized into Kubernetes Secrets without storing plaintext secrets in manifests.

---


