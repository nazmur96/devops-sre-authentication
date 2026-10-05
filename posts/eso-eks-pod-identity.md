---
title: "Getting AWS Secrets into EKS Pods with ESO and Pod Identity"
description: How External Secrets Operator and EKS Pod Identity let an app read a Secrets Manager secret without holding any AWS credentials, plus a step-by-step template.
crosspost:
  devto: full
  linkedin: summary
  tags: [kubernetes, aws, security, devops]
  hashtags: [Kubernetes, AWS]
  canonical_url: https://github.com/nazmur96/devops-sre-authentication/blob/main/posts/eso-eks-pod-identity.md
  summary: |
    Your pod needs a database password that lives in AWS Secrets Manager. The clean answer: the app never talks to AWS at all. It reads a plain Kubernetes Secret, and a middleman fills that Secret for it. That middleman is External Secrets Operator (ESO), and EKS Pod Identity gives ESO its AWS access.

    Think of it as a bank vault. Secrets Manager is the vault. ESO is a courier you hire. The Pod Identity Agent is the office that issues ID cards. The IAM role is the permission slip that says what the courier may pick up, and the Pod Identity association hands that slip to your courier.

    You tell the courier which bank (SecretStore) and which box (ExternalSecret). It picks up the secret, puts it in a locker inside your building (a normal K8s Secret), and checks back on a schedule. The app just opens the locker: no IAM role, no access keys, no idea AWS exists.

    Three things that bite: restart ESO after creating the association, because Pod Identity wires credentials only when a pod starts. Scope the IAM policy to just the secrets ESO needs. And env vars are read once, so a rotated secret needs a pod restart, a volume mount, or a tool like Reloader.

    I wrote it up step by step, with the IAM policies, the ClusterSecretStore and ExternalSecret YAML, verification commands and a troubleshooting list:
---

# Getting AWS Secrets into EKS Pods with ESO and Pod Identity

So we have an app/microservice/pod which needs access to a secret that we
stored in AWS Secrets Manager, let's say the database password.

We don't want our app to talk to AWS at all. We want the app to just find the
password inside Kubernetes as a normal K8s Secret. So we bring in a middleman
to fetch it for us: **ESO** (External Secrets Operator).

## Part 1: Setup (done once)

### Step 1: Install ESO in the cluster

We install External Secrets Operator, usually with Helm, into its own namespace
(`external-secrets`). This creates the ESO pods and a ServiceAccount for them,
also called `external-secrets`. Now we have a courier living in our cluster,
but it has no AWS ID card yet.

```text
EKS
 ├── App pod
 └── ESO pod  (ServiceAccount: external-secrets)
```

### Step 2: Install the Pod Identity Agent

We install the EKS Pod Identity Agent add-on. It runs a small helper on every
node. Its job is to hand out AWS credentials to pods that are allowed to have
them. Think of it as the ID card office.

### Step 3: Create an IAM role for ESO

We create an IAM role, say `ESO-Secrets-Reader`:

- **Permissions policy:** "you can read the secret `my-app/database`"
  (`GetSecretValue`, `DescribeSecret`, limited to only the secrets ESO needs).
- **Trust policy:** "I trust EKS Pod Identity (`pods.eks.amazonaws.com`) to
  hand me out to pods."

### Step 4: Connect the role to ESO's ServiceAccount

We create a Pod Identity association that tells EKS:

```text
namespace: external-secrets + ServiceAccount: external-secrets
                    ↓
            IAM role: ESO-Secrets-Reader
```

Meaning: "any pod running with this ServiceAccount gets this role." Now ESO has
an ID card.

> **Tip:** if ESO was already running before you created the association,
> restart the ESO pods. Pod Identity sets up the credential wiring when a pod
> starts, so old pods won't pick it up.

## Part 2: Telling ESO what to do

### Step 5: Create a SecretStore

This tells ESO **where** to look: "AWS Secrets Manager, region
`eu-central-1`." There are no access keys in here. ESO uses its Pod Identity
credentials automatically.

### Step 6: Create an ExternalSecret

This tells ESO **what** to fetch: "Get `my-app/database` from that store,
create a K8s Secret called `database-secret` in my app's namespace, and check
for changes every hour."

## Part 3: What happens automatically

### Step 7: ESO gets its credentials

ESO needs to call AWS, so the AWS SDK inside ESO asks the Pod Identity Agent
for credentials. The agent checks the association, gets temporary credentials
for `ESO-Secrets-Reader`, and gives them to ESO. They expire and get refreshed
automatically. No access keys are stored anywhere.

### Step 8: ESO fetches the secret

ESO calls Secrets Manager with those credentials and gets the password.

### Step 9: ESO creates the K8s Secret

ESO creates a normal Kubernetes Secret called `database-secret`. If someone
changes the password in AWS later, ESO notices on its next refresh and updates
it.

### Step 10: The app reads the secret

The app's YAML just says "give me `DB_PASSWORD` from Secret `database-secret`,"
as an env variable or a file. The app has no IAM role, no AWS credentials, and
no idea AWS exists.

## The picture to remember

```text
Pod Identity Agent ──gives temp creds──► ESO pod
                                          │
                                          │ fetch
                                          ▼
                                   Secrets Manager
                                          │
                                          │ password
                                          ▼
                                    ESO creates
                                          │
                                          ▼
                                 K8s Secret (database-secret)
                                          │
                                          │ reads
                                          ▼
                                       App pod
```

## The whole thing in one line

Install ESO and the Pod Identity Agent → give ESO an AWS role through a Pod
Identity association → tell ESO where to look (SecretStore) and what to fetch
(ExternalSecret) → ESO copies the secret into a K8s Secret → the app reads it.

## The analogy

Secrets Manager is a bank vault. ESO is a courier we hire (Step 1). The Pod
Identity Agent is the office that issues ID cards (Step 2). The IAM role is the
permission slip that says what the courier can pick up (Step 3), and the
association assigns that slip to our courier (Step 4). We tell the courier
which bank (SecretStore) and which box (ExternalSecret). The courier picks up
the cash and puts it in a locker inside our building (the K8s Secret), and our
app just opens the locker.

---

# Hands-on: step-by-step template

**Goal:** an app in EKS reads a secret from AWS Secrets Manager without having
any AWS credentials itself.

**Prerequisites:** a running EKS cluster, plus `aws`, `kubectl`, and `helm`
installed and configured.

### Step 0: Set your variables

```bash
export CLUSTER_NAME=my-cluster
export AWS_REGION=eu-central-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

### Step 1: Create a secret in AWS Secrets Manager

This is the secret our app needs.

```bash
aws secretsmanager create-secret \
  --name my-app/database \
  --region $AWS_REGION \
  --secret-string '{"username":"admin","password":"S3cr3t!"}'
```

### Step 2: Install ESO (the courier)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

This creates the ESO pods and a ServiceAccount named `external-secrets`.

### Step 3: Install the Pod Identity Agent (the ID card office)

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name eks-pod-identity-agent
```

Check that it's running on every node:

```bash
kubectl get daemonset eks-pod-identity-agent -n kube-system
```

### Step 4: Create an IAM role for ESO (the permission slip)

**Trust policy:** lets EKS Pod Identity hand this role to pods.

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
EOF

aws iam create-role \
  --role-name ESO-Secrets-Reader \
  --assume-role-policy-document file://trust-policy.json
```

**Permissions policy:** allows reading only our app's secrets.

```bash
cat > permissions-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:${AWS_REGION}:${ACCOUNT_ID}:secret:my-app/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ESO-Secrets-Reader \
  --policy-name read-my-app-secrets \
  --policy-document file://permissions-policy.json
```

If your secret is encrypted with a custom KMS key instead of the AWS default,
also allow `kms:Decrypt` on that key.

### Step 5: Connect the role to ESO's ServiceAccount (give the courier the slip)

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace external-secrets \
  --service-account external-secrets \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/ESO-Secrets-Reader
```

Then restart ESO so its pods pick up the new identity. Pods only get the
credential wiring when they start.

```bash
kubectl rollout restart deployment -n external-secrets
```

### Step 6: Tell ESO where to look (ClusterSecretStore)

```yaml
# cluster-secret-store.yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1   # change to your region
```

There's no `auth` section and no access keys. ESO automatically uses the
credentials Pod Identity gives it.

```bash
kubectl apply -f cluster-secret-store.yaml
```

If you're on an older ESO version, use `external-secrets.io/v1beta1` instead of
`v1`.

### Step 7: Tell ESO what to fetch (ExternalSecret)

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: database-secret
  namespace: my-app
spec:
  refreshInterval: 1h            # how often ESO checks AWS for changes
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: database-secret        # the K8s Secret ESO will create
  data:
    - secretKey: username        # key name inside the K8s Secret
      remoteRef:
        key: my-app/database     # secret name in AWS
        property: username       # JSON field inside it
    - secretKey: password
      remoteRef:
        key: my-app/database
        property: password
```

```bash
kubectl create namespace my-app
kubectl apply -f external-secret.yaml
```

### Step 8: The app reads the K8s Secret (opens the locker)

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: busybox
          command: ["sh", "-c", "echo App started; sleep 3600"]
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: database-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: database-secret
                  key: password
```

```bash
kubectl apply -f app.yaml
```

The app has no IAM role and no AWS credentials. It just reads a normal K8s
Secret.

### Step 9: Verify it works

```bash
# 1. Did ESO sync the secret? Look for STATUS = SecretSynced, READY = True
kubectl get externalsecret -n my-app

# 2. Does the K8s Secret exist with the right value?
kubectl get secret database-secret -n my-app \
  -o jsonpath='{.data.password}' | base64 -d

# 3. Does the app see it?
kubectl exec -n my-app deploy/my-app -- printenv DB_USERNAME
```

## Troubleshooting

- **ExternalSecret shows `AccessDenied`:** check the association exists
  (`aws eks list-pod-identity-associations --cluster-name $CLUSTER_NAME`),
  confirm the namespace and ServiceAccount names match exactly, and restart the
  ESO pods.
- **`ResourceNotFound`:** the secret name or region in AWS doesn't match
  `remoteRef.key` or the store's region.
- **Secret changed in AWS but the app still has the old value:** ESO updated
  the K8s Secret, but env vars are only read when a pod starts. Restart the
  app, mount the secret as a volume instead, or use a tool like Reloader to
  restart pods automatically.

## Cleanup

```bash
kubectl delete namespace my-app
helm uninstall external-secrets -n external-secrets
aws eks delete-pod-identity-association --cluster-name $CLUSTER_NAME \
  --association-id <id-from-list-command>
aws iam delete-role-policy --role-name ESO-Secrets-Reader --policy-name read-my-app-secrets
aws iam delete-role --role-name ESO-Secrets-Reader
aws secretsmanager delete-secret --secret-id my-app/database --force-delete-without-recovery
```
