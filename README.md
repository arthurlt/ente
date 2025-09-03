# What is Ente?
Ente (en-_tay_) is an open-source and fully end-to-end encrypted platform. Ente Photos should be thought of as an alternative to Google Photos. This guide will focus on Ente Photos, but because they both use the same server component you're free to also use Ente Auth app to store your 2FA codes.

# Goal
You can use the base manifests with your own overlays following the pattern documented [Off The Shelf Application](https://kubectl.docs.kubernetes.io/guides/config_management/offtheshelf/)

This will guide you through an opinionated way to run your own self-hosted instance of Ente on an existing Kubernetes cluster. You can also use the Kustomize base and  
## Prerequisites/Assumptions:
- Custom domain
    - With Fastmail setup
    - With Cloudflare DNS setup with a wildcard (`*.ente.<your_domain>`) to your Kubernetes Ingress IP
- A Cloudflare account
- A 1Password account
- A Fastmail account
- These tools:
    - `bash`
    - `kubectl`
    - `npm`
    - `op`[^3]
    - `git`
- A Kubernetes cluster with:
    - Any Ingress Controller
    - CloudNativePG[^4]
    - External Secrets Operator[^5]
    - Local Path Provisioner[^6]
    - cert-manager[^7]

# Configuring Secrets
## Create the vault
We'll want to create a vault we can share with our service account the External Secrets Operator will use.
```bash
op vault create Kubernetes \
    --description 'Secrets shared with External Secrets Operator in Kubernetes' \
    --icon application
```

## Create the service account
We'll need to create a service account for External Secrets Operator to access our Kubernetes vault in 1Password.
> [!NOTE]
> This assumes you installed External Secrets Operator in the `external-secrets` namespace on your Kubernetes cluster. 
```bash
kubectl create secret generic onepassword-token \
    --namespace external-secrets \
    --from-literal=token=$(op service-account create Kubernetes --vault Kubernetes:read_items --raw)
kubectl apply --namespace external-secrets --filename 'https://raw.githubusercontent.com/arthurlt/ente/main/docs/files/cluster-secret-store.yaml'
```
> [!CAUTION]
> The `ClusterSecretStore` can be referenced by `ExternalSecrets` in all namespaces in the cluster.

## Create Ente secrets
Ente provides default values for these, but we need to generate our own. Ente's official self-hosting quickstart script uses these same one-liners to generate a key, hash, and JWT secret.[^8]
```bash
op item create \
    --vault Kubernetes \
    --category Password \
    --title 'Ente Secrets' - \
    key[password]=$(head -c 32 /dev/urandom | base64 | tr -d '\n') \
    hash[password]=$(head -c 64 /dev/urandom | base64 | tr -d '\n') \
    jwt[password]=$(head -c 32 /dev/urandom | base64 | tr -d '\n' | tr '+/' '-_')
```

# Configuring Storage
Let's start by configuring our primary object store and database backup buckets. This guide will use Cloudflare's R2 service. If you know what you're doing or are open to some possible trial-and-error you're able to substitute with any S3-compatible service.

## Install Wrangler
Open a terminal, install Cloudflare's Wrangler CLI tool, then login.
```bash
npm install --global wrangler
wrangler login
```
> [!NOTE]
> If you see something like: `wrangler: command not found...`
> Then you will need to add npm's global prefix to your `PATH`. To do so for this session:
> ```bash
> export PATH=$(npm prefix --global)/bin:$PATH
> ```

## Create storage buckets
We'll use `wrangler` to create a bucket named 'ente' and configure it's CORS policy[^1][^2] then create a bucket for our database backups.
> [!CAUTION]
> It appears Ente sends a `null` value in the `Origin` header and required allowing all origins.
```bash
wrangler r2 bucket create ente
cat << 'EOF' > /tmp/cors.json
{
    "rules": [
        {
            "allowed": {
                "methods": ["GET", "HEAD", "POST", "PUT", "DELETE"],
                "origins": ["*"],
                "headers": ["*"]
            },
            "exposeHeaders": ["Etag"],
            "maxAgeSeconds": 3000
        }
    ]
}
EOF
wrangler r2 bucket cors set ente --file /tmp/cors.json
wrangler r2 bucket create cnpg
```


## Create R2 API tokens
We'll need to create API tokens and store them in 1Password.
1. Login to the [Cloudflare dashboard](https://dash.cloudflare.com/).
2. Select 'R2 object storage' on the left sidebar.
3. From the overview page select the dropdown labeled '{} API'.
4. Select 'Manage API tokens'.
5. From the token management page, select 'Create Account API token'.
6. Fill-out the token creation form for your Ente bucket.
    1. Input `ente` for the name.
    2. Select 'Object Read & Write' for the token permissions.
    3. Check 'Apply to specific buckets only' and choose your Ente bucket from the dropdown.
    4. Select 'Create Account API Token'.
    5. Save the values in 1Password, **replacing the variables with their values**.
        ```bash
        op item create \
            --vault Kubernetes \
            --category 'API Credential' \
            --title 'R2 - ente' - \
            'access key[password]'=$ACCESS_KEY_ID \
            'secret key[password]'=$SECRET_ACCESS_KEY \
            endpoint[url]=$DEFAULT_ENDPOINT \
            token[password]=$TOKEN_VALUE
        # remove the date fields (optional)
        op item edit 'R2 - ente' \
            --vault Kubernetes \
            expires[delete] \
            'valid from[delete]'
        ```
    6. Select 'Finish'.
7. Select 'Create Account API token' again.
8. Fill-out the token creation form for your CNPG bucket.
    9. Input `cnpg` for  the name.
    10. Select 'Object Read & Write' for the token permissions.
    11. Check 'Apply to specific buckets only' and choose your CNPG bucket from the dropdown.
    12. Select 'Create Account API Token'.
    13. Save the values in 1Password, **replacing the variables with their values**.
        ```bash
        op item create \
            --vault Kubernetes \
            --category 'API Credential' \
            --title 'R2 - cpng' - \
            'access key[password]'=$ACCESS_KEY_ID \
            'secret key[password]'=$SECRET_ACCESS_KEY \
            endpoint[url]=$DEFAULT_ENDPOINT \
            token[password]=$TOKEN_VALUE
        # remove the date fields (optional)
        op item edit 'R2 - cnpg' \
            --vault Kubernetes \
            expires[delete] \
            'valid from[delete]'
        ```
    14. Select 'Finish'.

# Configuring Email
We'll use Fastmail to handle SMTP. It should already be configured to send/receive from the domain you plan you use.
> [!NOTE]
> Ente uses email extensively for communicating to users and while it's not required to function, I highly recommend it.

## Create App Password
Fastmail requires an app password to authenticate with any third-party client. 
1. Login to Fastmail and navigate to your [Settings](https://app.fastmail.com/settings/theme).
2. Select 'Privacy & Security' under 'Account' on the left sidebar.
3. Select 'Manage app passwords and access' near the bottom.
4. Under 'App passwords', select '+ New app password'.
5. Fill out the new app password form.
    1. Select the 'Name' dropdown, choose 'Custom...', and enter `ente`.
    2. Select the 'Access' dropdown and select 'SMTP'.
    3. Select 'Generate password'.
6. Save the value in 1Password, **replacing the variables with their values**.
    ```bash
    op item create \
        --vault Kubernetes \
        --category 'Email Account' \
        --title 'Fastmail - ente' - \
        SMTP.username=$FASTMAIL_LOGIN \
        SMTP.password=$APP_PASSWORD \
        SMTP.'SMTP Server'=smtp.fastmail.com \
        SMTP.'port number'=587 \
        SMTP.security=TLS \
        SMTP.'auth method'=Password
    ```
> [!IMPORTANT]
> The username will be the email/username **you** login to Fastmail with, not the email Ente will be using to send.

# Configuring Certificates
We'll be using cert-manager, Let's Encrypt, and Cloudflare DNS to automatically provision and rotate TLS certificates. 

## Create Cloudflare API Token
We'll solve Let's Encrypt's ACME DNS01 challenge with Cloudflare. 
1. Login to the [Cloudflare dashboard](https://dash.cloudflare.com/).
2. Select 'Manage Account' then 'Account API tokens' on the left sidebar.
3. Select 'Create Token'
4. Under 'API token templates' find 'Edit zone DNS' and select 'Use template'.
    1. Select the pencil icon next to 'Token name' and input `cert-manager`.
    2. Under 'Permissions' select '+ Add more'.
    3. For the second permissions row select `Zone`, `Zone`, and `Read`.
    4. Under 'Zone Resources' select `Include`, `All zones from an account`, and select your account from the last dropdown. 
    5. Select 'Continue to summary'.
    6. Select 'Create Token'.
    7. Save the value in 1Password, **replacing the variable with the token**.
        ```bash
        op item create \
            --vault Kubernetes \
            --category 'API Credential' \
            --title 'Cloudflare - cert-manager' - \
            credential=$API_TOKEN \
            type='Bearer Token'
        # remove the date fields (optional)
        op item edit 'Cloudflare - cert-manager' \
            --vault Kubernetes \
            expires[delete] \
            'valid from[delete]'
        ```

## Create ExternalSecret
(TODO)
> [!NOTE]
> This assumes you installed cert-manager in the `cert-manager` namespace on your Kubernetes cluster. 
```bash
kubectl apply --namespace cert-manager --filename 'https://raw.githubusercontent.com/arthurlt/ente/main/docs/files/external-secret.yaml'
```

# Replace
(TODO)
Replace the values the Homelab overlay, **replacing the variable with your domain**.
```bash
git clone https://github.com/arthurlt/ente.git
sed --in-place 's/arthur.computer/$CUSTOM_DOMAIN/g' ente/overlays/homelab/*.yaml
sed --in-place 's|https://f061e3d9b3cb2a466cdbef9ea19cee24.r2.cloudflarestorage.com|$R2_ENDPOINT|g' ente/overlays/homelab/*.yaml
```

---
Caused by: Error validating origin
= the webauthn 

---
# Sources:
help.ente.io/self-hosting/installation/env-var
https://help.ente.io/self-hosting/installation/config
https://help.ente.io/self-hosting/administration/object-storage
https://help.ente.io/self-hosting/installation/post-install
https://help.ente.io/self-hosting/administration/cli
https://developers.cloudflare.com/r2/buckets/cors
https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/
https://kubectl.docs.kubernetes.io/references/kustomize/builtins/
https://kubectl.docs.kubernetes.io/guides/config_management/components/
https://cloudnative-pg.io/documentation/1.27/applications/
https://cloudnative-pg.io/documentation/1.27/samples/

---
# TODO
- Cilium network policies component
- Add initContainer to museum to wait for DB
- Cleanup 1Password CLI stuff (templates?)
- Create cert-manager component?
- Create External Secret Operator component?

I intend to write future guides on:
- Setting up a Kubernetes cluster using Talos Linux
- Installing CloudNativePG on your K8s cluster
- Installing cert-manager on your K8s cluster and configuring with Cloudflare DNS
- Installing external secrets operator on your K8s cluster and configuring with 1Password
- Setting up [Garage](https://garagehq.deuxfleurs.fr/) for object storage (and migrating to it)

[^1]: https://help.ente.io/self-hosting/administration/object-storage#cors-cross-origin-resource-sharing

[^2]: https://developers.cloudflare.com/api/resources/r2/subresources/buckets/subresources/cors/methods/update/

[^3]: https://developer.1password.com/docs/cli/get-started/

[^4]: https://cloudnative-pg.io/

[^5]: https://external-secrets.io/latest/

[^6]: https://github.com/rancher/local-path-provisioner

[^7]: https://cert-manager.io/

[^8]: https://github.com/ente-io/ente/blob/main/server/quickstart.sh
