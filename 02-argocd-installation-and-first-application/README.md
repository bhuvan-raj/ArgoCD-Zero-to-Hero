# ArgoCD Installation and Setup

## 1. Install Argo CD on Kubernetes

Step 1: Create a namespace

```
kubectl create namespace argocd
```

Step 2: Install Argo CD

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs Argo CD components in the `argocd` namespace, including:

- Argo CD API Server
- Argo CD Repository Server
- Argo CD Application Controller
- Argo CD Redis
- Argo CD Dex Server
- Argo CD Application Set Controller

Step 3: Verify the installation

```
kubectl get pods -n argocd
```

Check the services:

```
kubectl get svc -n argocd
```

Wait until the required pods are in the `Running` state.

## 2. Access the Argo CD UI

For initial access, use port forwarding.

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open your browser:

```
https://localhost:8080
```

Accept the self-signed certificate warning if prompted.

## 3. Get the initial admin password

The default username is:

```
admin
```

The initial password is stored in the Kubernetes secret named `argocd-initial-admin-secret`.

### Linux (Bash)

Option 1: Using kubectl

```
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Option 2: Using Argo CD CLI

```
argocd admin initial-password -n argocd
```

### Windows (PowerShell)

Option 1: Using kubectl

```
$secret = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"

[System.Text.Encoding]::UTF8.GetString(
    [System.Convert]::FromBase64String($secret)
)
```

Option 2: Using Argo CD CLI

```
argocd admin initial-password -n argocd
```

Both approaches retrieve the same initial password.

## 4. Login to Argo CD

Open:

```
https://localhost:8080
```

Use these credentials:

| Field    | Value                      |
| -------- | -------------------------- |
| Username | `admin`                    |
| Password | Retrieved initial password |

## 5. Change the default password

After logging in, change the default password.

Using Argo CD CLI:

```
argocd account update-password
```

After changing it, delete the initial password secret:

```
kubectl delete secret argocd-initial-admin-secret -n argocd
```

This is recommended for security.

## 6. Useful verification commands

```
# Check all Argo CD pods
kubectl get pods -n argocd

# Check services
kubectl get svc -n argocd

# Check deployments
kubectl get deployments -n argocd

# Check Argo CD applications
kubectl get applications -n argocd

# Check Argo CD secrets
kubectl get secrets -n argocd
```

## Installation workflow

1. Create namespace

   `kubectl create namespace argocd`
2. Install Argo CD

   Apply the official installation manifest.
3. Verify pods

   `kubectl get pods -n argocd`
4. Expose UI

   Port-forward `argocd-server`.
5. Retrieve password

   Use the Linux or PowerShell command.
6. Login and change password

   Login as `admin` and update the password.

Important: The commands above are for a standard, non-HA installation. For production, pin a specific Argo CD version and consider the HA installation.

Official documentation: [Argo CD Getting Started](https://github.com/argoproj/argo-cd/blob/master/docs/getting_started.md?plain=1\&utm_source=chatgpt.com) · [Argo CD CLI Installation](https://github.com/argoproj/argo-cd/blob/master/docs/cli_installation.md?utm_source=chatgpt.com).
