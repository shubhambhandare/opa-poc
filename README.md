# 🛡️ Open Policy Agent (OPA) - Gatekeeper PoC  

## 📌 Overview  

This **Proof of Concept (PoC)** demonstrates the use of **Open Policy Agent (OPA) - Gatekeeper** to enforce security policies in Kubernetes.  

### 🔹 **How It Works**  
- **Gatekeeper** is installed to enforce admission control policies.  
- We define **three security policies** (**ConstraintTemplates**) to restrict Kubernetes Pods based on:  
  1. **Non-root user enforcement**  
  2. **Allowed container image registries**  
  3. **Required resource requests/limits**  
- Each policy has:  
  - A **ConstraintTemplate** (policy logic).  
  - A **Constraint** (enforcement rule).  
  - **Example Pods** that are either **compliant** (allowed) or **non-compliant** (denied).  
- **Pods violating the policies will be rejected**, while compliant Pods will be scheduled.  
- **Policies build on each other**, meaning previous policies remain active when testing new use cases.  

---

## 🛠 Steps to Execute  

### 1️⃣ Install Gatekeeper Using Helm  

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
```

---

## 🚀 Use Cases & Policy Enforcement  

### 🔹 **Use Case 1: Enforcing Non-Root Containers**  

#### ✅ **ConstraintTemplate: RootContainers**  

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: rootcontainers
spec:
  crd:
    spec:
      names:
        kind: RootContainers
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package root_containers

        import future.keywords.in

        violation[{"msg": msg}] {
          some container in input.review.object.spec.containers
          not container.securityContext.runAsUser
          msg := "Container must have runAsUser set"
        }

        violation[{"msg": msg}] {
          some container in input.review.object.spec.containers
          container.securityContext.runAsUser == 0
          msg := sprintf("Root user (UID 0) is not allowed in container %s", [container.name])
        }
```

#### ✅ **Constraint: Enforce Non-Root Policy**  

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RootContainers
metadata:
  name: no-root-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
  parameters: {}
```

#### ❌ **Example: Non-Compliant Pod (Denied)**  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod-1
  labels:
    team_id: nginx
    app: root-container
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        runAsUser: 0
        allowPrivilegeEscalation: true
```

📌 **Reason:** The Pod runs as `root` (UID 0), which is not allowed.  

---

