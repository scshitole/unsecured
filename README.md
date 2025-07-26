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

Would you like me to zip this up into a deployable IRSA + pod example for practice or demo?

