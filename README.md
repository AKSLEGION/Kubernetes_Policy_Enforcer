# 🚧 Kubernetes Policy Enforcer

**Goal:**  
Ensure Kubernetes security compliance by enforcing policies at deployment time using **OPA Gatekeeper**.

---

## 🔒 Introduction

Modern containerized applications need strong runtime and deployment security. Kubernetes by default is flexible, but lacks built-in enforcement of custom organizational policies like:

- Blocking privileged containers.
- Enforcing label presence.
- Restricting hostPath volumes.

This project demonstrates **how to use OPA Gatekeeper** with Kubernetes to enforce such policies dynamically and block non-compliant workloads.

---

## 🧱 Architecture

```
┌─────────────────────────────┐
│        Kubernetes API       │
├─────────────────────────────┤
│   Admission Controller      │◄──────┐
├─────────────────────────────┤       │
│        OPA Gatekeeper       │       │
├─────────────────────────────┤       │
│     ConstraintTemplates     │       │
│         Constraints         │       │
└─────────────────────────────┘       │
                                      │
┌─────────────────────────────┐       │
│     User kubectl apply      │───────┘
└─────────────────────────────┘
```

- **OPA Gatekeeper**: Validates resources on admission.
- **Admission Controller**: Intercepts `kubectl apply` requests.
- **Policy Store**: Holds templates (Rego) and constraints (rules).
- **Monitoring**: Prometheus integration for policy evaluation metrics.

---

## ⚙️ Implementation Steps

### 1. Install Minikube & kubectl (in WSL2)
Follow steps provided in `install_minikube.sh` or manually install:
```bash
minikube start --driver=docker
kubectl version --client
```

### 2. Deploy Gatekeeper
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml
```

Wait for pods:
```bash
kubectl get pods -n gatekeeper-system
```

---

### 3. Write a Policy (ConstraintTemplate)

This template blocks containers with `privileged: true`.

**File:** `templates/k8srequiredprivileged.yaml`
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container '%v' is not allowed.", [container.name])
        }
```

---

### 4. Create a Constraint

**File:** `constraints/deny-privileged.yaml`
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredPrivileged
metadata:
  name: disallow-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Apply both:
```bash
kubectl apply -f templates/k8srequiredprivileged.yaml
kubectl apply -f constraints/deny-privileged.yaml
```

---

### 5. Test the Policy

**Privileged Pod Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: badcontainer
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
```

```bash
kubectl apply -f test-pods/privileged-pod.yaml
# Should be rejected with a violation message.
```

**Valid Pod Example:**
```bash
kubectl run safe-pod --image=nginx --restart=Never
# Should be accepted.
```

---

## 📈 Monitoring

Gatekeeper exposes metrics at port `8888` (Prometheus scrape target). You can view:

- Total policy evaluations
- Violations per constraint
- Admission decisions

Optionally integrate with Grafana dashboards for real-time policy observability.

---

## 📦 Project Structure

```
kubernetes-policy-enforcer/
├── templates/
│   └── k8srequiredprivileged.yaml
├── constraints/
│   └── deny-privileged.yaml
├── test-pods/
│   └── privileged-pod.yaml
├── README.md
└── install_minikube.sh (optional)
```

---

## ✅ Conclusion

- Policies are enforced **before** resources reach the cluster.
- Prevents insecure workloads in real-time.
- Fully extensible to enforce other security/compliance rules.

This system can be integrated into CI/CD pipelines to ensure DevSecOps from day one.

---