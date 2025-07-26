

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
