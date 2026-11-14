::: {align="center"}
# ☸️ Kubernetes Multi-Namespace RBAC & Application Deployment

### Two independent applications · Two namespaces · Separate identities and permissions

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35.9-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![RBAC](https://img.shields.io/badge/Security-RBAC-2E7D32?style=for-the-badge)
![Isolation](https://img.shields.io/badge/Design-Namespace%20Isolation-6A5ACD?style=for-the-badge)

**Objective:** Onboard two technically separate applications with
independent Kubernetes namespaces, ServiceAccounts, Roles, RoleBindings,
Deployments, and Services.
:::

------------------------------------------------------------------------

## 📚 Contents

1.  [Architecture and Design](#1--architecture-and-design)
2.  [Resource Naming](#2--resource-naming)
3.  [How RBAC Is Applied](#3--how-rbac-is-applied)
4.  [Debit Card Application --- Namespace and
    RBAC](#4--debit-card-application--namespace-and-rbac)
5.  [Credit Card Application --- Namespace and
    RBAC](#5--credit-card-application--namespace-and-rbac)
6.  [Validate Access Boundaries](#6--validate-access-boundaries)
7.  [Deploy Debit Card Application](#7--deploy-debit-card-application)
8.  [Deploy Credit Card Application](#8--deploy-credit-card-application)
9.  [Validate Both Applications](#9--validate-both-applications)
10. [Expose the Applications with Services](#10--expose-the-applications-with-services)
11. [Troubleshooting](#11--troubleshooting)
12. [Cleanup](#12--cleanup)
13. [Production Considerations](#13--production-considerations)

------------------------------------------------------------------------

## 1. 🧭 Architecture and Design

The Debit Card and Credit Card applications are treated as **two
independent application workloads**. Each gets its own namespace and its
own RBAC objects.

```{=html}
<pre>
                    Kubernetes Cluster
                           |
             +-------------+-------------+
             |                           |
             v                           v
      debit-card-lab              credit-card-lab
       (Namespace)                 (Namespace)
             |                           |
      debit-card-deployer         credit-card-deployer
       (ServiceAccount)             (ServiceAccount)
             |                           |
      debit-card-role             credit-card-role
          (Role)                      (Role)
             |                           |
      debit-card-binding         credit-card-binding
       (RoleBinding)                (RoleBinding)
             |                           |
      Debit Card App              Credit Card App
       Deployment                    Deployment
             |                           |
      debit-card-service         credit-card-service
         (Service)                    (Service)
</pre>
```
### Separation principle

  Component        Debit Card              Credit Card
  ---------------- ----------------------- ------------------------
  Namespace        `debit-card-lab`        `credit-card-lab`
  ServiceAccount   `debit-card-deployer`   `credit-card-deployer`
  Role             `debit-card-role`       `credit-card-role`
  RoleBinding      `debit-card-binding`    `credit-card-binding`
  Deployment       `debit-card-app`        `credit-card-app`
  Service          `debit-card-service`    `credit-card-service`

**No shared application namespace, deployment identity, Role, or
RoleBinding.**

------------------------------------------------------------------------

## 2. 🗂️ Resource Naming

All resources are created independently. The Role and RoleBinding are
namespace-scoped Kubernetes objects.

  Resource         Debit Card              Credit Card
  ---------------- ----------------------- ------------------------
  Namespace        `debit-card-lab`        `credit-card-lab`
  ServiceAccount   `debit-card-deployer`   `credit-card-deployer`
  Role             `debit-card-role`       `credit-card-role`
  RoleBinding      `debit-card-binding`    `credit-card-binding`
  Deployment       `debit-card-app`        `credit-card-app`
  Service          `debit-card-service`    `credit-card-service`

------------------------------------------------------------------------

## 3. 🔐 How RBAC Is Applied

-   **Namespace:** Provides a logical boundary for namespaced resources.
-   **ServiceAccount:** Provides an identity for automation that
    authenticates to the Kubernetes API.
-   **Role:** Defines permissions within one namespace.
-   **RoleBinding:** Grants that Role to a subject, such as the
    namespace's ServiceAccount.

Each application has its own RBAC chain:

``` text
Debit pipeline identity
  -> debit-card-deployer
  -> debit-card-binding
  -> debit-card-role
  -> debit-card-lab only

Credit pipeline identity
  -> credit-card-deployer
  -> credit-card-binding
  -> credit-card-role
  -> credit-card-lab only
```

> \[!IMPORTANT\] A RoleBinding grants permissions to its ServiceAccount;
> it does not automatically make every application Pod use that
> identity. In this runbook, the ServiceAccounts represent deployment
> automation identities. The sample application Pods do not need
> Kubernetes API permissions and therefore do not use these deployer
> ServiceAccounts.

------------------------------------------------------------------------

## 4. 💳 Debit Card Application --- Namespace and RBAC

Run administrator commands from the control-plane VM or another
authorized Kubernetes client.

### 4.1 Create the namespace

``` bash
kubectl create namespace debit-card-lab
```

Verify:

``` bash
kubectl get namespace debit-card-lab
```

### 4.2 Create the ServiceAccount

``` bash
kubectl create serviceaccount debit-card-deployer -n debit-card-lab
```

Verify:

``` bash
kubectl get serviceaccount -n debit-card-lab
```

### 4.3 Create the Role

Create `debit-card-role.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: debit-card-role
  namespace: debit-card-lab
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply:

``` bash
kubectl apply -f debit-card-role.yaml
```

### 4.4 Create the RoleBinding

Create `debit-card-binding.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: debit-card-binding
  namespace: debit-card-lab
subjects:
  - kind: ServiceAccount
    name: debit-card-deployer
    namespace: debit-card-lab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: debit-card-role
```

Apply:

``` bash
kubectl apply -f debit-card-binding.yaml
```

### 4.5 Verify Debit Card RBAC

``` bash
kubectl get serviceaccount,role,rolebinding -n debit-card-lab
```

------------------------------------------------------------------------

## 5. 💳 Credit Card Application --- Namespace and RBAC

The Credit Card application gets its own independent set of Kubernetes
resources.

### 5.1 Create the namespace

``` bash
kubectl create namespace credit-card-lab
```

Verify:

``` bash
kubectl get namespace credit-card-lab
```

### 5.2 Create the ServiceAccount

``` bash
kubectl create serviceaccount credit-card-deployer -n credit-card-lab
```

Verify:

``` bash
kubectl get serviceaccount -n credit-card-lab
```

### 5.3 Create the Role

Create `credit-card-role.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: credit-card-role
  namespace: credit-card-lab
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Apply:

``` bash
kubectl apply -f credit-card-role.yaml
```

### 5.4 Create the RoleBinding

Create `credit-card-binding.yaml`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: credit-card-binding
  namespace: credit-card-lab
subjects:
  - kind: ServiceAccount
    name: credit-card-deployer
    namespace: credit-card-lab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: credit-card-role
```

Apply:

``` bash
kubectl apply -f credit-card-binding.yaml
```

### 5.5 Verify Credit Card RBAC

``` bash
kubectl get serviceaccount,role,rolebinding -n credit-card-lab
```

------------------------------------------------------------------------

## 6. ✅ Validate Access Boundaries

These checks are run by a cluster administrator. The `--as` option
impersonates the ServiceAccount for authorization testing; the
administrator needs permission to impersonate that identity.

### Debit Card identity --- allowed in its own namespace

``` bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:debit-card-lab:debit-card-deployer \
  -n debit-card-lab
```

Expected: `yes`

### Debit Card identity --- denied in Credit Card namespace

``` bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:debit-card-lab:debit-card-deployer \
  -n credit-card-lab
```

Expected: `no`

### Credit Card identity --- allowed in its own namespace

``` bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:credit-card-lab:credit-card-deployer \
  -n credit-card-lab
```

Expected: `yes`

### Credit Card identity --- denied in Debit Card namespace

``` bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:credit-card-lab:credit-card-deployer \
  -n debit-card-lab
```

Expected: `no`

### Secret access --- not granted by either example Role

``` bash
kubectl auth can-i get secrets \
  --as=system:serviceaccount:debit-card-lab:debit-card-deployer \
  -n debit-card-lab
```

Expected: `no`

The same check can be performed for the Credit Card identity.

------------------------------------------------------------------------

## 7. 🚀 Deploy Debit Card Application

This example uses NGINX as a placeholder web workload. It demonstrates
Kubernetes deployment and service creation; it is not a real debit-card
business application.

Create `debit-card-app.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: debit-card-app
  namespace: debit-card-lab
  labels:
    app: debit-card-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: debit-card-app
  template:
    metadata:
      labels:
        app: debit-card-app
    spec:
      containers:
        - name: debit-card-app
          image: nginx:stable-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: debit-card-service
  namespace: debit-card-lab
spec:
  type: NodePort
  selector:
    app: debit-card-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30081
```

Apply:

``` bash
kubectl apply -f debit-card-app.yaml
```

------------------------------------------------------------------------

## 8. 🚀 Deploy Credit Card Application

Create `credit-card-app.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: credit-card-app
  namespace: credit-card-lab
  labels:
    app: credit-card-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: credit-card-app
  template:
    metadata:
      labels:
        app: credit-card-app
    spec:
      containers:
        - name: credit-card-app
          image: nginx:stable-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: credit-card-service
  namespace: credit-card-lab
spec:
  type: NodePort
  selector:
    app: credit-card-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30082
```

Apply:

``` bash
kubectl apply -f credit-card-app.yaml
```

------------------------------------------------------------------------

## 9. 🔍 Validate Both Applications

### Check each namespace independently

``` bash
kubectl get all -n debit-card-lab
```

``` bash
kubectl get all -n credit-card-lab
```

### Check deployment rollout

``` bash
kubectl rollout status deployment/debit-card-app -n debit-card-lab
kubectl rollout status deployment/credit-card-app -n credit-card-lab
```

### Check Pods and node placement

``` bash
kubectl get pods -n debit-card-lab -o wide
kubectl get pods -n credit-card-lab -o wide
```

### Check Services

``` bash
kubectl get services -n debit-card-lab
kubectl get services -n credit-card-lab
```

Expected deployment state in each namespace:

``` text
NAME             READY   UP-TO-DATE   AVAILABLE
<application>    2/2     2            2
```

------------------------------------------------------------------------

## 10. 🌐 Expose the Applications with Services

This lab uses Kubernetes **NodePort Services** so users can access the sample applications through a VM node IP and a published node port. No port-forwarding is used.

NodePort is suitable for this local lab demonstration. In production, applications are commonly exposed through an Ingress controller or a cloud/provider LoadBalancer, depending on the platform architecture.

### Debit Card Service

The Service in `debit-card-app.yaml` uses:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: debit-card-service
  namespace: debit-card-lab
spec:
  type: NodePort
  selector:
    app: debit-card-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30081
```

### Credit Card Service

The Service in `credit-card-app.yaml` uses:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: credit-card-service
  namespace: credit-card-lab
spec:
  type: NodePort
  selector:
    app: credit-card-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30082
```

Apply the manifests:

```bash
kubectl apply -f debit-card-app.yaml
kubectl apply -f credit-card-app.yaml
```

### Verify Services

```bash
kubectl get services -n debit-card-lab
kubectl get services -n credit-card-lab
```

Expected service types and ports:

- Debit Card: `NodePort`, `80:30081/TCP`
- Credit Card: `NodePort`, `80:30082/TCP`

### User access

From a machine that can reach the VirtualBox host-only network:

| Application | URL |
|---|---|
| Debit Card | `http://192.168.56.5:30081` |
| Credit Card | `http://192.168.56.5:30082` |

NodePort is available on each node address, provided that the node is reachable:

- `http://192.168.56.6:30081`
- `http://192.168.56.6:30082`

If a page is unreachable, check VM network reachability, the guest firewall, Service selectors, Pod readiness, and Service endpoints.

> [!NOTE]
> NodePort exposes a service through a port on each node. It does not provide an application hostname, TLS termination, or an external load balancer. Those are typically handled by an Ingress controller or a LoadBalancer Service in a production design.

---

## 11. 🧰 Troubleshooting

  -----------------------------------------------------------------------------------------------
  Symptom                             Diagnostic command
  ----------------------------------- -----------------------------------------------------------
  Pod is Pending                      `kubectl describe pod <pod> -n <namespace>`

  ImagePullBackOff                    `kubectl describe pod <pod> -n <namespace>`

  CrashLoopBackOff                    `kubectl logs <pod> -n <namespace>`

  Deployment not ready                `kubectl describe deployment <deployment> -n <namespace>`

  Service has no endpoints            `kubectl get endpoints -n <namespace>`

  Forbidden error                     Check the namespace, Role, RoleBinding, resource, and verb
  -----------------------------------------------------------------------------------------------

### Review events

``` bash
kubectl get events -n debit-card-lab --sort-by=.lastTimestamp
kubectl get events -n credit-card-lab --sort-by=.lastTimestamp
```

### Review application logs

``` bash
kubectl logs deployment/debit-card-app -n debit-card-lab
kubectl logs deployment/credit-card-app -n credit-card-lab
```

------------------------------------------------------------------------

## 12. 🧹 Cleanup

Delete the application resources independently:

``` bash
kubectl delete -f debit-card-app.yaml
kubectl delete -f credit-card-app.yaml
```

Delete Debit Card RBAC:

``` bash
kubectl delete rolebinding debit-card-binding -n debit-card-lab
kubectl delete role debit-card-role -n debit-card-lab
kubectl delete serviceaccount debit-card-deployer -n debit-card-lab
```

Delete Credit Card RBAC:

``` bash
kubectl delete rolebinding credit-card-binding -n credit-card-lab
kubectl delete role credit-card-role -n credit-card-lab
kubectl delete serviceaccount credit-card-deployer -n credit-card-lab
```

Delete namespaces only when the entire corresponding lab is no longer
required:

``` bash
kubectl delete namespace debit-card-lab
kubectl delete namespace credit-card-lab
```

> \[!CAUTION\] Deleting a namespace removes all namespaced resources
> inside it. Confirm the target namespace before running the command.

------------------------------------------------------------------------

## 13. 🏭 Production Considerations

1.  **Keep application boundaries separate.** Each application owns its
    namespace and namespaced resources.
2.  **Use separate deployment identities.** Do not share a
    ServiceAccount between unrelated application pipelines.
3.  **Apply least privilege.** Remove permissions not needed by the
    deployment process. For example, remove `delete` if it is not
    required.
4.  **Keep runtime identities separate.** Application Pods should not
    inherit deployment automation permissions.
5.  **Protect credentials.** Do not commit ServiceAccount tokens or
    kubeconfig files to Git. Prefer approved CI/CD identity integrations
    and short-lived credentials.
6.  **Control Secrets.** Neither example Role grants Secret access. Add
    narrowly scoped access only when there is a real requirement.
7.  **Understand namespace boundaries.** Namespaces and RBAC provide
    logical and authorization isolation, but they do not by themselves
    enforce network isolation. Use NetworkPolicies and appropriate CNI
    enforcement when network separation is required.
8.  **Set resource governance.** Consider ResourceQuota and LimitRange
    objects for each application namespace.
9.  **Audit changes.** Track which deployment identity changed which
    application resources.

------------------------------------------------------------------------

::: {align="center"}
### Two Applications · Two Namespaces · Two RBAC Boundaries

**Separate ownership. Separate deployment identities. Controlled
access.**

*Local Kubernetes multi-namespace deployment lab.*
:::
