# ✅ Goal

**Only allow members of a specific RBAC group (e.g., `debug-team`) to create pods that use an image containing `nettools`.**
Everyone else gets a validation failure.

Which ValidatingAdmissionPolicy (VAP) + ValidatingAdmissionPolicyBinding

Blocks any Pod spec containing a forbidden image (e.g., nettools, debug, etc.).

---

# 🧩 1. ValidatingAdmissionPolicy — Block Pods Using Forbidden Debug Images

This policy denies any workload type that results in Pods containing a debug image:

- Pod
- Deployment
- ReplicaSet
- StatefulSet
- DaemonSet
- Job
- CronJob
- kubectl debug ephemeral containers

It works because VAPs evaluate the final Pod spec after mutating webhooks (Webhooks or policies that might change your YAML e.g., adding default values), validates the request before it is persisted to the database (etcd) and before any controller sees it.

If a request fails validation, it is rejected immediately by the API Server, and the Controller never even knows the request happened.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-nettools-debug-image
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods", "pods/ephemeralcontainers"] # pods/ephemeralcontainers will prevent user attach a debug container to existing pod
  variables:
    - name: isNettoolsImage
      expression: >
        object.spec.containers.exists(c, c.image.contains("nettools")) ||
        (
          object.spec.initContainers != null &&
          object.spec.initContainers.exists(c, c.image.contains("nettools"))
        ) ||
        (
          object.spec.ephemeralContainers != null &&
          object.spec.ephemeralContainers.exists(c, c.image.contains("nettools"))
        )
    - name: hasAccess
      expression: request.userInfo.groups.exists(g, g == "debug-team")
  validations:
    - expression: "!(variables.isNettoolsImage) || variables.hasAccess"
      message: "Only users in group 'debug-team' can create pods using nettools images."
```

### Explanation

- `isNettoolsImage`: checks **all containers** (including init containers/ephemeralContainers) for an image containing `"nettools"`.
- `hasAccess`: checks if user belongs to the Kubernetes group `debug-team`.
- Validation only fails when:

  - the pod uses a nettools image **and**
  - the user **is not** in the allowed group.

---

# 🧩 2. Binding (ValidatingAdmissionPolicyBinding)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: restrict-nettools-debug-image-binding
spec:
  policyName: restrict-nettools-debug-image
  validationActions: ["Deny", "Audit"]
```

# 🧩 3. RBAC

Suppose this is on DEV environment, both group has the same permission (to make sure the devs-team not be blocked by RBAC but VAPs)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: debug-team-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/attach"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/ephemeralcontainers"]
    verbs: ["patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: debug-team-binding
subjects:
  - kind: Group
    name: debug-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: debug-team-role
  apiGroup: rbac.authorization.k8s.io
---
# For normal user
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: devs-team-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/attach"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/ephemeralcontainers"]
    verbs: ["patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devs-team-binding
subjects:
  - kind: Group
    name: devs-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: devs-team-role
  apiGroup: rbac.authorization.k8s.io
```

# ✅ 4. Create two new users in Minikube

Minikube uses **client certificate authentication** by default.
We’ll create:

- **debug-user** → belongs to group `debug-team`
- **test-user** → belongs to devs-team group (should be blocked by VAP)

---

# 🧩 4.1 Create certificates for each user

### Generate keys

```bash
openssl genrsa -out debug-user.key 2048
openssl genrsa -out test-user.key 2048
```

### Generate certificate signing requests

```bash
openssl req -new -key debug-user.key -out debug-user.csr -subj "/CN=debug-user/O=debug-team"
openssl req -new -key test-user.key -out test-user.csr -subj "/CN=test-user/O=devs-team"
```

Note:

- `CN` = Kubernetes username
- `O` = Kubernetes group

---

# 🧩 4.2 Sign the certificates using Minikube's CA

Minikube stores the CA files here:

```
~/.minikube/ca.crt
~/.minikube/ca.key
```

Run the signing:

```bash
openssl x509 -req -in user-minikube/debug-user.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out user-minikube/debug-user.crt -days 365
openssl x509 -req -in user-minikube/test-user.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out user-minikube/test-user.crt -days 365
```

---

# 🧩 4.3 Create kubeconfig entries for both users

### Add credentials

```bash
kubectl config set-credentials debug-user \
  --client-certificate=user-minikube/debug-user.crt \
  --client-key=user-minikube/debug-user.key

kubectl config set-credentials test-user \
  --client-certificate=user-minikube/est-user.crt \
  --client-key=user-minikube/test-user.key
```

### Create contexts (pointing to Minikube cluster)

```bash
kubectl config set-context debug-user-context \
  --cluster=minikube \
  --user=debug-user

kubectl config set-context test-user-context \
  --cluster=minikube \
  --user=test-user
```

---

# 🧪 Test (example)

### A normal user:

Suppose still stay at `minikube context` (no need to switch the context)

```bash
kubectl --context=test-user-context run test --image=nettools:optimize
```

❌ Should fail with:

> The pods "test" is invalid: : ValidatingAdmissionPolicy 'restrict-nettools-debug-image' with binding 'restrict-nettools-debug-image-binding' denied request: Only users in group 'debug-team' can create pods using nettools images.

Moreover, if try to attach a debug container to existing pod

```bash
kubectl debug --context=test-user-context -it productpage-v1-bb87ff47b-2r4wt --image=nettools:optimize --target=productpage
```

❌ Should fail also: Only users in group 'debug-team' can create pods using nettools images.

### A user in `debug-team` group:

```bash
kubectl --context=debug-user-context run test --image=nettools:optimize
```

✅ Should be allowed.

> pod/test created

---

# 🛡️ Security Scanning

To ensure the image is free from known vulnerabilities, you can scan it using **Trivy**.

### Run Trivy via Docker

If you don't have Trivy installed locally, you can run it using Docker:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image nettools:optimize
```

### Expected Output (Clean)

```
2025-11-21T13:04:17Z    INFO    [vuln] Vulnerability scanning is enabled
2025-11-21T13:04:17Z    INFO    [secret] Secret scanning is enabled
2025-11-21T13:04:17Z    INFO    [secret] If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2025-11-21T13:04:17Z    INFO    [secret] Please see https://trivy.dev/v0.67/docs/scanner/secret#recommendation for faster secret detection
2025-11-21T13:04:18Z    INFO    Detected OS     family="alpine" version="3.20.8"
2025-11-21T13:04:18Z    INFO    [alpine] Detecting vulnerabilities...  os_version="3.20" repository="3.20" pkg_num=68
2025-11-21T13:04:18Z    INFO    Number of language-specific files      num=0

Report Summary

┌───────────────────────────────────┬────────┬─────────────────┬─────────┐
│              Target               │  Type  │ Vulnerabilities │ Secrets │
├───────────────────────────────────┼────────┼─────────────────┼─────────┤
│ nettools:optimize (alpine 3.20.8) │ alpine │        0        │    -    │
└───────────────────────────────────┴────────┴─────────────────┴─────────┘
Legend:
- '-': Not scanned
- '0': Clean (no security findings detected)
```
