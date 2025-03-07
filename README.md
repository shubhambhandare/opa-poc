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

### 🔹 **Use Case 2: Restricting Allowed Container Registries**  

#### ✅ **ConstraintTemplate: AllowedRepos**  

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedrepos
spec:
  crd:
    spec:
      names:
        kind: AllowedRepos
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowed_repos

        import future.keywords.in

        allowed_registry = "gcr.io"

        violation[{"msg": msg}] {
          input.review.object.metadata.namespace != "kube-system"
          some container in input.review.object.spec.containers
          not startswith(container.image, allowed_registry)
          msg := sprintf("Container image %s is not from the allowed registry (%s)", [container.image, allowed_registry])
        }
```

#### ✅ **Constraint: Enforce Allowed Registries**  

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRepos
metadata:
  name: enforce-gcr-registry
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
  parameters: {}
```

#### ✅ **Example: Compliant Pod (Allowed)**  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: allowed-pod
  namespace: default
  labels:
    team_id: sre
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: my-container
      image: gcr.io/my-project/my-image:latest
      securityContext:
        runAsUser: 1000
      command: ["sleep", "3600"]
```

📌 **Reason:** The container image originates from the allowed registry (`gcr.io`).  

---

### 🔹 **Use Case 3: Enforcing `team_id` Label Requirement**  

#### ✅ **ConstraintTemplate: TeamId**  

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: teamid
spec:
  crd:
    spec:
      names:
        kind: TeamId
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package team_id_label

        import future.keywords.if

        default allow := false

        allow if input.review.object.metadata.labels.team_id

        violation[{"msg": msg, "details": {}}] {
          not allow
          msg := "The team_id label is required"
        }
```

#### ✅ **Constraint: Enforce `team_id` Label**  

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: TeamId
metadata:
  name: teamid-pods
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
  name: invalid-pod
spec:
  containers:
    - name: nginx
      image: nginx:alpine
```

📌 **Reason:** The `team_id` label is missing.  

#### ✅ **Example: Compliant Pod (Allowed)**  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: valid-pod
  labels:
    team_id: frontend
spec:
  containers:
    - name: nginx
      image: gcr.io/nginx:alpine
      securityContext:
        runAsUser: 1011
```

📌 **Reason:** The `team_id` label is present, making the pod compliant.  

---

