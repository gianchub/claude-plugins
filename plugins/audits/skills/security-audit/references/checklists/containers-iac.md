# Containers and Infrastructure-as-Code

## Scope

Dockerfiles, docker-compose, Kubernetes manifests, Helm charts, Terraform, CloudFormation, Pulumi, Ansible, and equivalent infrastructure-defining code. Cloud-provider-specific configurations (IAM policies, security group definitions, S3/GCS bucket policies). Infrastructure misconfigurations are a top breach class for cloud-deployed applications; the controls are often easier to fix than application code but require knowing where to look.

## Dockerfile

### User Identity

- **`USER root` (or no `USER`) at runtime** — Container runs as root inside the namespace. Combined with privilege escalation in the runtime, escapes are easier. Set `USER nonroot` (or a numeric UID) for non-init workloads.
- **Numeric UID 0 from a named user** — Some "non-root" users are still UID 0; verify with `id` in the image or check `/etc/passwd`.
- **Root file ownership** — Files copied as root, then run as nonroot, may have permission issues; intentional pattern is `COPY --chown=nonroot:nonroot ...`.
- **Capabilities** — Default capabilities in Docker include `NET_RAW`, `SYS_CHROOT` etc.; for tighter security drop all and add only what's needed (often nothing).

### Base Image

- **Minimal images** — `distroless`, `alpine`, `slim` variants reduce attack surface and CVE exposure.
- **Pin tag and digest** — `FROM node:18.17.0@sha256:abc...` for reproducibility and CVE-tracking.
- **EOL distros** — Old Ubuntu / Alpine / Debian releases without security updates.
- **Multi-stage builds** — Final stage should be minimal; build tools (compilers, package managers, shells beyond what's needed) shouldn't be in the runtime image.

### Secrets and Sensitive Data

- **`ENV SECRET=...`** — Persisted in image layers; visible in `docker history`.
- **`ARG` for secrets** — Visible in image metadata if not removed.
- **`COPY .env`, `COPY config/`** — Secrets directly in image.
- **Build-time secret mounts** — Use Docker BuildKit `--mount=type=secret`; secrets not persisted in layers.
- **Verify with `docker history` and tools like `dive`** — Identify secrets in any layer.

### Filesystem and Permissions

- **`chmod 777` / `chmod -R 777`** — World-writable; flag.
- **World-writable directories** — `/tmp` etc. are typically OK; app-owned directories should be `0750` or tighter.
- **SUID/SGID binaries** — Removed from image where not needed (`find / -perm /6000 -type f`).
- **Read-only root filesystem** — `--read-only` flag at runtime; image designed for it (writable mounts only where needed). Strong defense.

### Networking

- **Exposed ports** — `EXPOSE` is documentation, not a control; runtime port mapping enforces. Documents intended exposure surface.
- **Privileged ports** — Binding to ports < 1024 traditionally requires root; non-root with capability `NET_BIND_SERVICE` is the alternative.

### Entrypoint and Command

- **`ENTRYPOINT`** preferred over `CMD` for forcing the entry; `CMD` for defaults.
- **`USER` after `WORKDIR`** — User must have permission on the workdir.
- **Init process** — Long-running services without an init handler may zombie children; `tini` / `dumb-init` helps; image entrypoint in a shell loop is a finding.
- **Shell as ENTRYPOINT** — `ENTRYPOINT ["bash", "-c", "..."]` with user-controllable argv may yield command injection.

### Healthcheck

- `HEALTHCHECK` defined; meaningful (curl localhost / actual app endpoint, not `true`).
- Defensive: healthcheck doesn't expose internals.

### Build Hygiene

- `.dockerignore` excludes `.git`, `.env`, secrets, large unrelated files. Without it, build context may include sensitive data and make image bloated.
- Layer caching considerations: COPY `package.json` before COPY `.` for npm caching.

## docker-compose

- **Privileged containers** — `privileged: true` grants all capabilities; rarely needed; flag.
- **Volume mounts** — `:/var/run/docker.sock` mounting Docker socket grants container full Docker control = host root.
- **Host networking** — `network_mode: host` removes container network isolation.
- **Sensitive bind mounts** — Mounting `/`, `/etc`, `/var/run` from host.
- **Default credentials** — Database services with `MYSQL_ROOT_PASSWORD: password`, etc.; flag in production-intended files.
- **Environment files** — `env_file: .env` referencing committed `.env` with secrets.
- **Networks** — Verify networks segment as expected.

## Kubernetes Manifests

### Pod Security

- **`securityContext`** at pod and container level:
  - `runAsNonRoot: true`
  - `runAsUser:` non-zero
  - `readOnlyRootFilesystem: true`
  - `allowPrivilegeEscalation: false`
  - `capabilities.drop: ["ALL"]` and add only required
  - `privileged: false`
  - `seccompProfile.type: RuntimeDefault` (or stricter)
- **`hostNetwork`, `hostPID`, `hostIPC`** — All `false` unless needed.
- **`hostPath` mounts** — Filesystem from host into container; flag.
- **AppArmor / SELinux** — Annotations applying profiles; defense-in-depth.

### Network Policies

- **`NetworkPolicy` resources** — Default-deny ingress and egress per namespace; explicit allow rules for required traffic. Without policies, all pods can talk to all pods (depending on CNI).
- **Egress control** — Block egress to internal Kubernetes services, cloud metadata, public internet where not needed.

### Service Accounts

- **Default service account** — Pods using default SA may have unintended permissions; bind specific SA per pod.
- **`automountServiceAccountToken: false`** — Pods that don't need k8s API access shouldn't have a token mounted.
- **`Role` / `ClusterRole` permissions** — Audit RBAC for over-permissive bindings (`*` on `*` resources, especially at cluster scope).
- **`ClusterRoleBinding` to default SA** — Catastrophic; flag.

### Secrets

- **Secret resources with literal data** — `data` field in committed manifests is base64-encoded, not encrypted; treat as plaintext. Use `SealedSecrets`, `External Secrets Operator`, or external secret stores.
- **Mount path** — Secrets mounted as files preferred over env vars (env vars more easily leak in logs / dumps).
- **In-cluster encryption at rest** — etcd encryption configuration; document if absent.

### Image Pull and Admission

- **`imagePullPolicy: Always`** with tag-only references doesn't guarantee reproducibility; pin to digest.
- **`imagePullSecrets`** for private registries; verify rotation.
- **Admission controllers** — OPA / Gatekeeper / Kyverno enforcing security policies; document if used or recommend if absent.
- **Pod Security Admission (PSA)** — Built-in admission control for security contexts. `restricted` profile is the strong default.

### Other K8s

- **Ingress configurations** — TLS termination, cipher suites, allowed paths. Permissive `pathType: Prefix` with backend missing authn = exposed.
- **`emptyDir.sizeLimit`** — Without limit, attacker can fill node disk via pod.
- **Resource limits** — `resources.limits.cpu`, `resources.limits.memory`; absence enables noisy-neighbor and resource-exhaustion DoS.

## Helm

- **Hardcoded secrets in `values.yaml`** — Same as committed manifests; flag.
- **`values.schema.json`** — Schema validation; recommend if absent.
- **Subchart trust** — Subcharts pulled from registries; verify and pin.

## Terraform

### General

- **State files** — `.tfstate` may contain plaintext secrets (output values, sensitive resource attributes). Never commit; remote state in S3/GCS/Azure with encryption-at-rest and access controls.
- **`*.tfvars`** — Variable values; may contain secrets; treat as sensitive. `*.tfvars.example` should have placeholders only.
- **Module sources** — Public modules from registry; pin to version; review trustworthiness of unfamiliar maintainers.
- **Provider version pinning** — `required_providers` block; pin to specific versions for reproducibility.

### AWS Resources (common)

- **S3 bucket public access** — `public_access_block` resource configured to block all public access; `acl = "private"`; bucket policy not allowing `*` principal.
- **S3 bucket encryption** — `server_side_encryption_configuration` with KMS or SSE-S3.
- **S3 bucket versioning** — Enabled for important data.
- **IAM policies** — `Action: ["*"]`, `Resource: ["*"]` are over-permissive; iterate towards least privilege.
- **Security groups** — Inbound `0.0.0.0/0` on non-public-app ports (SSH 22, RDP 3389, DB ports); flag.
- **Default VPC use** — Custom VPC with explicit subnets / route tables preferred.
- **EBS encryption** — Default-on for new accounts; verify.
- **RDS public access** — `publicly_accessible = false` for production databases.
- **RDS encryption** — `storage_encrypted = true`.
- **Lambda environment variables with secrets** — Use SSM Parameter Store / Secrets Manager.
- **CloudTrail** — Enabled for audit; multi-region trail recommended.
- **GuardDuty** / **Security Hub** — Enabled at organization level.

### GCP Resources

- **GCS bucket** — `uniform_bucket_level_access` enabled; not publicly accessible unless intended.
- **IAM bindings** — `roles/owner` on resources should be rare; `roles/editor` and `roles/viewer` are coarse, prefer custom roles.
- **VPC firewall rules** — Default-deny; explicit allow lists.
- **GKE cluster** — Private cluster; node SA with minimal permissions; workload identity for pod-to-GCP-API.

### Azure Resources

- **Storage account** — `allow_blob_public_access = false`; firewall rules restrict source IPs.
- **Network Security Groups** — Same as AWS security groups.
- **Key Vault** — RBAC mode; access policies tightly scoped.
- **Activity Log** — Retention configured; alerting on suspicious operations.

## CloudFormation / Pulumi / CDK

Same security domains as Terraform; framework idioms differ but the underlying cloud configuration is what's audited. Cloud-provider-agnostic findings (IAM wildcards, public buckets, open SGs) apply identically.

## Common IaC Findings

### Permissions and IAM

- **`*` in policy actions** — Without strong scope on resources, full access.
- **Wildcard resource ARNs** — `Resource: ["arn:aws:s3:::*/*"]` allows all buckets; least privilege scopes to specific bucket.
- **`AssumeRole` from any account** — `Principal: {"AWS": "*"}` on a trust policy.
- **Long-lived access keys** — Prefer roles assumed by workload identity; flag any user with active access keys for service-to-service.
- **Service account / instance role attached unnecessarily** — Compute resources attached to roles broader than they need.

### Network

- **Open SSH/RDP** — From `0.0.0.0/0`; should be from bastion / VPN only.
- **Database ports open** — `5432`, `3306`, `1433`, `27017`, `6379` open to internet.
- **Open all ports / protocols** — `protocol: -1` and `cidr: 0.0.0.0/0` together.
- **Missing egress restrictions** — Default cloud SGs allow all egress.

### Storage

- **Public buckets** — S3, GCS, Azure Blob with public-read/write ACLs or bucket policies.
- **Unencrypted storage** — At rest.
- **No versioning / no MFA-delete on critical buckets** — Ransomware mitigation; recommend for important data.

### Compute

- **Public-facing compute without WAF / load balancer** — Direct exposure of EC2 / VM to internet.
- **SSH key authentication only** — No password auth (good); but key management is a separate concern.
- **Out-of-date AMIs / images** — Snapshot images stale; security patches not applied.

### Secrets

- **Hardcoded in `*.tf`, `*.tfvars`, `*.yaml`** — Same as application secrets; covered in `secrets-and-keys.md`.
- **Secret manager integration missing** — Using env vars directly instead of fetching from KMS/SecretsManager/Vault.

### Logging and Monitoring

- **Audit logs not enabled** — CloudTrail / Audit Logs / Activity Log; should be on at the org level.
- **Logs not retained** — Compliance lifetime missing.
- **Alarms not configured** — On security-relevant events (root login, IAM changes, security group changes).

## Specific Tools and Linters

Mention as recommendations (Claude-only analysis means we can't run them):

- **Trivy** — Scan IaC, container images, dependencies for vulns.
- **Checkov** — IaC security scanner.
- **tfsec / kics** — Terraform-specific.
- **Kubescape / kubesec / kube-linter** — Kubernetes manifest scanners.
- **dockle / hadolint** — Dockerfile linters.
- **Polaris** — Kubernetes best practices.

## Recommendation Patterns

- Pin container base images to digests; rebuild on cadence.
- Run as non-root; use distroless or minimal base images; drop capabilities.
- Apply Pod Security Admission `restricted` profile in Kubernetes.
- NetworkPolicy for default-deny + explicit allow.
- IAM least privilege; replace `*` policies with resource- and action-scoped equivalents.
- Encrypt storage at rest; restrict public access on object storage.
- Use secret managers; never commit secrets to IaC.
- Centralize cloud audit logging; alert on security-relevant operations.
- Run IaC scanners in CI; treat findings as blocking for merge.
