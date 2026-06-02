Step 1 — Create and activate a Python virtual environment

```bash
python3 -m venv ~/tap-venv
source ~/tap-venv/bin/activate
```
Step 2 — Install Python dependencies

```bash
pip install --upgrade pip
pip install ansible boto3 botocore kubernetes
```

Step 3 — Install required Ansible collections

```bash
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.general
```

Step 4 — Gather: OpenShift token
You need a token for a user with cluster-admin privileges.

```bash
# Option A — if already logged in via oc
oc login --server=https://api.mycluster.domain.com:6443

# Get the token for the current session
oc whoami --show-token
# → sha256~vFanQbt...
```
Save the output — that is your token.

Step 5 — Gather: OpenShift server URL

```bash
oc cluster-info | grep "Kubernetes control plane"
# → https://api.mycluster.domain.com:6443
```

That URL is your server.

Step 6 — Gather: GitHub Personal Access Token (RHDH only)

1. Go to → GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens (or classic)

2. Create a token with the following scopes (classic PAT):
  * repo (full)
  * read:org
  * read:user
  * user:email

3. Copy the token — that is your `github_pat`

Step 7 — Gather: AWS credentials (TPA only)

The playbook needs programmatic access to create an S3 bucket in us-east-2.

Option A — use an existing IAM user:

AWS Console → IAM → Users → <your user> → Security credentials → Create access key
Select "Other" as the use case → Create → copy Access key ID and Secret access key.

Option B — create a minimal IAM user:

AWS Console → IAM → Users → Create user
  Username: tap-tpa-s3
  → Attach policy directly: AmazonS3FullAccess (or a scoped policy)
  → Create user → Security credentials → Create access key
The Access key ID = aws_ec2_access_key
The Secret access key = aws_ec2_secret_key

Step 8 — Set environment variables
```bash
export TOKEN="sha256~vFanQbt..."
export SERVER="https://api.mycluster.domain.com:6443"
export GITHUB_PAT="github_pat_..."
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="wJalrXUtn..."
```

Step 9 — Run the playbook

```bash
cd ansible

# Full install (all components)
ansible-playbook \
  -e token=$TOKEN \
  -e server=$SERVER \
  -e github_pat=$GITHUB_PAT \
  -e aws_ec2_access_key=$AWS_ACCESS_KEY_ID \
  -e aws_ec2_secret_key=$AWS_SECRET_ACCESS_KEY \
  playbook.yml
```

Step 10 — Deactivate the venv when done
```bash
deactivate
```
---
Logging into Developer Hub as Admin

How authentication works

Dev Hub is configured to use Keycloak OIDC — there is no local Dev Hub login. All users authenticate through Keycloak's rhdh realm. The RBAC maps Keycloak groups to Dev Hub roles:

Keycloak Group	Dev Hub Role	Can add templates?
- rhdh-admins	role:default/admin	✅ Yes
- rhdh-devs	role:default/dev	✅ Yes (limited)

Step 1 — Get the Dev Hub URL

```bash
oc get route backstage-developer-hub -n rhdh -o jsonpath='{.spec.host}'
```
Open https://<that-host> in your browser.

Step 2 — Log in with the admin user

Click Sign In → you'll be redirected to Keycloak. Use:

Field	Value
Username	`admin`
Password	`password`

This is the `keycloak_user_username` / `keycloak_user_password` from `keycloak/defaults/main.yml`. The admin user is in the `rhdh-admins` Keycloak group, which maps to role:default/admin in Dev Hub.

There is also a pre-created developer user if needed (note use this, as it has more admin permissions than `admin`):

Field	Value
Username	`dev1`
Password	`developer`

Step 3 — Add templates (two options)

Option A — Via the app-config ConfigMap (persistent, survives restarts):

Edit the ConfigMap to add your template URL:

```bash
oc edit configmap app-config-rhdh -n rhdh
```

Under `catalog.locations`, add an entry:

```yaml
catalog:
  locations:
    - rules:
      - allow:
        - Template
      target: https://github.com/your-org/your-repo/blob/main/templates.yaml
      type: url
```
Then restart Dev Hub to pick up the change:

```bash
oc rollout restart deployment/backstage-developer-hub -n rhdh
```

Option B — Via the Dev Hub UI (temporary, lost on restart):

Go to `Catalog` → `Create` (or the `+` icon)

Click `Register Existing Component`

Paste the URL to your `catalog-info.yaml` or `templates.yaml`

Click `Analyze` → `Import`