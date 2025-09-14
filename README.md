# AWX + Minikube SFTP Backup for Cisco UC (CUCM/IM&P/Unity)

This repo contains an Ansible playbook to deploy an **SFTP endpoint** inside your **Minikube** cluster for Cisco **DRS** backups. It creates:
- Namespace, PVC, Secret, ConfigMap
- SFTP Deployment (atmoz/sftp)
- NodePort Service on **30222**
- Optional NetworkPolicy restricted to your UC IPs
- Daily retention CronJob (delete files older than 14 days)

> **Credentials**: The SFTP credentials are **generated at runtime**. Do **not** store Cisco admin passwords in this repo.

## Files
- `deploy_sftp_for_cucm.yml` — main playbook
- `collections/requirements.yml` — Galaxy collections (auto-installed by AWX project sync if enabled)
- `inventory/hosts.ini` — optional local inventory (AWX inventory is configured in UI)
- `k8s/awx-rbac.yaml` — ServiceAccount + ClusterRoleBinding (cluster-admin) for AWX (adjust namespace if needed)

## 1) Push to GitHub
Create a new GitHub repo and push these files.

## 2) Allow AWX to manage the cluster
> AWX is running on Minikube. Use a ServiceAccount and a token-based K8s credential.

```bash
# If your AWX namespace is different, change -n awx accordingly.
kubectl apply -f k8s/awx-rbac.yaml

# For Kubernetes 1.24+ create a token for the SA:
kubectl -n awx create token awx-k8s-admin

# Get the cluster API endpoint:
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'; echo
```

In **AWX → Credentials → Add → Kubernetes/OpenShift API Bearer Token**:
- **Host**: the API server URL you printed above
- **Bearer Token**: output of `kubectl -n awx create token awx-k8s-admin`
- **Verify SSL**: provide CA if needed (optional for Minikube; you can disable verify for testing)

> For production, replace cluster-admin with a scoped Role/RoleBinding limited to the namespaces/resources you need.

## 3) Create AWX Project (SCM from GitHub)
- **Organization**: choose yours
- **Name**: `Cisco UC SFTP Project`
- **SCM Type**: Git
- **SCM URL**: your GitHub repo URL
- Enable:
  - **Update Revision on Launch**
  - **Allow Branch Override**
  - **Install content using Ansible Galaxy** (so `collections/requirements.yml` is installed)

## 4) Create AWX Inventory
- **Name**: `Local`
- Add a **Host**: `localhost` (no variables needed), or import from `inventory/hosts.ini` if you prefer.

## 5) Create AWX Job Template
- **Name**: `Deploy SFTP for Cisco UC`
- **Inventory**: `Local`
- **Project**: `Cisco UC SFTP Project`
- **Playbook**: `deploy_sftp_for_cucm.yml`
- **Credentials**: the **Kubernetes/OpenShift** credential you created
- **Verbosity**: 1 (or higher for debugging)

Optionally add a **Survey** to override variables like `pvc_size`, `retention_days`, or switch to `service_type: LoadBalancer` if you have MetalLB.

## 6) Run the Job
At the end of the run, AWX prints:
- SFTP **username**: `cucm`
- SFTP **password**: random 24-char
- **Host**: `$(minikube ip)`
- **Port**: `30222`
- **Directory**: `/upload`

## 7) Configure Cisco DRS
For each app (CUCM, IM&P, Unity Connection):
- Protocol: **SFTP**
- Host: output of `minikube ip`
- Port: **30222**
- User: **cucm**
- Password: from AWX job output
- Directory: **/upload**
- Test, then schedule backups.

## Notes
- **NetworkPolicy** requires a CNI that enforces policies (Calico, Cilium, etc.). If not present, the policy is ignored; connectivity still works via NodePort.
- To use an external IP instead of NodePort, install **MetalLB** in Minikube and set `service_type: LoadBalancer`.
- Offload archives to object storage by adding an `rclone`/`restic` CronJob mounting the same PVC.
