# 📘 Kubernetes RBAC – Pod-to-Pod Metrics Access

## 📌 Objective

Configure **RBAC** so that a **Prometheus (metrics-collector) pod** can:

- `get`
- `list`
- `watch`

**Pods inside the `monitoring` namespace only**, using:

- `Role`
- `ServiceAccount`
- `RoleBinding`

Following the **Least Privilege Principle**.

---

# 🟢 Step 1: Create Namespace

## 📄 `namespace.yml`

    apiVersion: v1
    kind: Namespace
    metadata:
      name: monitoring

### Apply:

    kubectl apply -f namespace.yml

---

# 🟢 Step 2: Create Role (Read-Only Pods Access)

## 📄 `role.yml`

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: metrics-read-role
      namespace: monitoring
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]

✔ Only read access to pods  
❌ Cannot create / delete / modify  

---

# 🟢 Step 3: Create ServiceAccount

## 📄 `serviceaccount.yml`

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: metrics-collector-sa
      namespace: monitoring

---

# 🟢 Step 4: Bind Role to ServiceAccount

## 📄 `rolebinding.yml`

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: metrics-read-binding
      namespace: monitoring
    subjects:
    - kind: ServiceAccount
      name: metrics-collector-sa
      namespace: monitoring
    roleRef:
      kind: Role
      name: metrics-read-role
      apiGroup: rbac.authorization.k8s.io

✔ Connects **Role → ServiceAccount**

---

# 🟢 Step 5: Deploy Node Exporter Pods

## 📄 `node-exporter.yml`

    apiVersion: v1
    kind: Pod
    metadata:
      name: node-exporter-1
      namespace: monitoring
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: node-exporter-2
      namespace: monitoring
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100

---

# 🟢 Step 6: Deploy Metrics Collector Pod

## 📄 `metrics-collector.yml`

    apiVersion: v1
    kind: Pod
    metadata:
      name: metrics-collector
      namespace: monitoring
    spec:
      serviceAccountName: metrics-collector-sa
      containers:
      - name: collector
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          echo "Starting metrics collector..."
          while true; do
            sleep 30
          done

✔ This pod uses the **ServiceAccount**  
✔ RBAC rules apply to this pod  

---

# 🚀 Apply All Resources

From inside the project folder:

    kubectl apply -f .

---

# 🔍 Verify Pods

    kubectl get pods -n monitoring -o wide

Example:

    metrics-collector   192.168.43.31
    node-exporter-1     192.168.14.252
    node-exporter-2     192.168.23.65

---

# 🟢 Test Metrics Access

Enter the `metrics-collector` pod:

    kubectl exec -it metrics-collector -n monitoring -- sh

Test connectivity using pod IPs:

    curl http://<node-exporter-1-ip>:9100/metrics | head -5
    curl http://<node-exporter-2-ip>:9100/metrics | head -5

### Expected output:

    # HELP go_gc_duration_seconds ...
    # TYPE go_gc_duration_seconds summary

---

# 🔐 Verify RBAC Permissions

### ✅ Check Allowed Action

    kubectl auth can-i list pods \
      --as=system:serviceaccount:monitoring:metrics-collector-sa \
      -n monitoring

Expected:

    yes

---

### ❌ Check Forbidden Action

    kubectl auth can-i delete pods \
      --as=system:serviceaccount:monitoring:metrics-collector-sa \
      -n monitoring

Expected:

    no

---

# ⚠ Important Concept

**RBAC controls Kubernetes API access — NOT network traffic.**

### RBAC Controls:

    kubectl get pods
    kubectl delete pods
    kubectl create pods

### RBAC Does NOT Control:

    curl http://pod-ip:9100

Pod-to-pod communication is handled by **cluster networking**, not RBAC.

---

# 🎯 Architecture Flow

    Role (get, list, watch pods)
            ↓
        RoleBinding
            ↓
      ServiceAccount
            ↓
     metrics-collector Pod
            ↓
    Reads node-exporter metrics

---

# 🎯 Interview Explanation

> I created a namespaced Role with `get`, `list`, and `watch` permissions on pods, bound it to a ServiceAccount using a RoleBinding, and attached that ServiceAccount to the Prometheus pod to enforce least privilege access within the `monitoring` namespace.
