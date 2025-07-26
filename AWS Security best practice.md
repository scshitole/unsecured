Perfect — let’s take **real examples** directly from the Wiz AWS Security Best Practices Cheat Sheet you uploaded, and show how **Wiz adds value** in each case.

---

## 🔍 1. **AWS Config (Agentless Configuration Scanning)**

> *Cheat Sheet says:* “Enable AWS Config to monitor resource configurations and changes without using agents.”

### 🔧 AWS Native:

* AWS Config tracks changes (e.g., S3 bucket becomes public).
* It shows history of config changes.

### 🔐 **How Wiz Helps:**

* **Wiz ingests AWS Config data** across accounts/regions.
* Correlates config changes with **runtime risk** (e.g., that public S3 bucket is linked to a pod with RDS access).
* Shows **real attack paths**, not just isolated events.
* Auto-detects **misconfigured VPCs**, security groups, IAM roles — and **maps them to exposure levels**.

🧠 *Wiz adds cross-resource context + prioritization.*

---

## 📦 2. **CloudTrail + Centralized Logging**

> *Cheat Sheet says:* “Use CloudTrail to log API calls and store in S3 securely.”

### 🔧 AWS Native:

* You must manually analyze CloudTrail logs (or plug them into GuardDuty).
* No built-in prioritization or context.

### 🔐 **How Wiz Helps:**

* Automatically **pulls CloudTrail logs** into its platform.
* Maps suspicious API activity to **real users, roles, assets**.
* Connects logs to **live risk paths** (e.g., “User X accessed S3 → IAM → EC2 with secrets”).

🧠 *Wiz brings intelligence and correlation to raw logs.*

---

## 🔐 3. **KMS – Data Encryption**

> *Cheat Sheet says:* “Use AWS KMS to encrypt data at rest (e.g., S3, EBS, RDS).”

### 🔧 AWS Native:

* You can check encryption status via Config or console.
* No context about data sensitivity.

### 🔐 **How Wiz Helps:**

* Finds **unencrypted resources** (S3, EBS, RDS, DynamoDB).
* Identifies **if they contain sensitive data** (e.g., PII, secrets).
* Prioritizes: “Unencrypted S3 bucket with public access + user token inside.”

🧠 *Wiz combines encryption, exposure, and data sensitivity.*

---

## 📊 4. **AWS Security Hub – CSPM**

> *Cheat Sheet says:* “Use Security Hub to aggregate GuardDuty, Inspector, Macie, etc.”

### 🔧 AWS Native:

* Security Hub consolidates findings but lacks deep correlation.
* Findings may be duplicated or low-priority.

### 🔐 **How Wiz Helps:**

* Replaces or **enhances CSPM** by:

  * Aggregating findings from AWS + Wiz's own scan engine
  * Correlating multiple risks into **a single exploit path**
  * Prioritizing based on actual impact (data, identity, external access)

🧠 *Wiz = CSPM + Runtime Graph + Risk Prioritization*

---

## ☁️ 5. **EKS Security with App Mesh**

> *Cheat Sheet says:* “Use AWS App Mesh to control intra-service traffic in EKS.”

### 🔧 AWS Native:

* You set up App Mesh manually for fine-grained control.
* No visibility into risky pods, privileges, or open ports.

### 🔐 **How Wiz Helps:**

* **Scans EKS clusters** for:

  * Privileged pods
  * Unrestricted service-to-service communication
  * Misconfigured Ingress or NodePorts
* **Maps traffic paths** to sensitive services or secrets

🧠 *Wiz finds broken mesh patterns & security gaps in K8s.*

---

## 📉 6. **Amazon Detective – Visualizing Security Findings**

> *Cheat Sheet says:* “Use Amazon Detective to visualize findings with ML + graph.”

### 🔧 AWS Native:

* Visualizes logs and patterns from CloudTrail/GuardDuty.
* Good for forensic investigation.

### 🔐 **How Wiz Helps:**

* Builds its own **dynamic security graph** across:

  * Identities
  * Workloads
  * Network
  * Data
* Provides **real-time attack path visualizations**
* Focuses on **prevention** and **prioritization**, not just forensics

🧠 *Wiz is proactive vs Detective's reactive visualizations.*

---

## 🌐 7. **CNAPP (Cloud-Native Application Protection Platform)**

> *Cheat Sheet says:* “Use CNAPP like Wiz for multi-cloud, full-stack security.”

### 🔧 AWS Native:

* Each AWS tool handles a **specific vertical** (IAM, data, network).
* Multi-cloud support is minimal or non-existent.

### 🔐 **How Wiz Helps:**

* **Unifies security visibility across AWS, Azure, GCP**
* Scans **everything** — VM, container, code, identities, S3, KMS, logs, etc.
* Helps you move from:

  * “I have 1000 alerts”
  * to
  * “Here are the **3 actual attack paths** that need immediate fixing.”

---

### ✅ Summary Table

| Cheat Sheet Best Practice | AWS Native Tool        | Wiz Value                                    |
| ------------------------- | ---------------------- | -------------------------------------------- |
| AWS Config                | Tracks changes         | Correlates misconfigs with exploit paths     |
| CloudTrail                | Logs activity          | Builds context around risky behavior         |
| KMS                       | Manages keys           | Flags unencrypted sensitive resources        |
| Security Hub              | Centralized alerts     | Prioritizes + deduplicates across accounts   |
| App Mesh                  | Traffic control in EKS | Detects bad pod configs & lateral movement   |
| Detective                 | Visualizes past alerts | Shows real-time attack paths                 |
| CNAPP                     | Not available natively | Wiz gives full-stack, multi-cloud visibility |

