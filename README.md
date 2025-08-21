# Network Policy in OpenShift

## 1. Overview

- NetworkPolicy is used to **isolate pods** from other pods within the same namespace or across different namespaces.
- It can be configured **without admin privileges**, giving developers more control over application traffic.
- It controls access **based on labels**, unlike traditional firewalls that rely on IP addresses.

---

## 2. Labeling a Namespace

To set a label on a namespace:

```bash
oc label namespace
```

For example, to label the `test` namespace:

```bash
oc label namespace test network=network-isolation
```

---

## 3. Deny All Traffic to Pods in a Namespace

### 3.1 Create Two Projects

```bash
oc new-project pol1
oc new-project pol2
```

### 3.2 Create Deployments in pol1

```bash
oc project pol1
oc create deployment httpd1-pol1 --image=registry.redhat.io/rhel8/httpd-24
oc create deployment httpd2-pol1 --image=registry.redhat.io/rhel8/httpd-24
```

### 3.3 Get Pod IPs

```bash
oc get pods -o wide
```

Sample output:
```
NAME                           READY   STATUS    RESTARTS   AGE    IP             NODE                  NOMINATED NODE   READINESS GATES
httpd1-pol1-776f659d5f-j82pv   1/1     Running   0          2m5s   10.131.0.232   worker-0.example.com   <none>           <none>
httpd2-pol1-67c7bff97b-q7g4p   1/1     Running   0          73s    10.131.0.233   worker-0.example.com   <none>           <none>
```

### 3.4 Test Connectivity (Expected: Success)

```bash
oc rsh httpd2-pol1-776f659d5f-j82pv
sh-4.4$ curl 10.131.0.232:8080
<head>
Output Omitted
</head>
```

### 3.5 Apply Deny-All Policy

Create `denyall.yaml`:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: denyall
spec:
  podSelector: {}
  ingress: []
```

Apply the policy:

```bash
oc apply -f denyall.yaml
```

### 3.6 Test Connectivity Again (Expected: Blocked)

```bash
oc rsh httpd2-pol1-776f659d5f-j82pv
sh-4.4$ curl 10.131.0.232:8080
^C
```

---

## 5. Allow Traffic from All Pods in pol1

### 5.1 Create Policy `allow-pol1.yaml`

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-pol1
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

### 5.2 Apply Policy

```bash
oc apply -f allow-pol1.yaml
```

### 5.3 Test Communication

```bash
oc rsh httpd2-pol1-776f659d5f-j82pv
sh-4.4$ curl 10.131.0.232:8080
<head>
Output Omitted
</head>
```

---

## 6. Allow Traffic from Specific Namespace and Pod

### 6.1 Create Deployments in pol2

```bash
oc project pol2
oc create deployment httpd1-pol2 --image=registry.redhat.io/rhel8/httpd-24
oc create deployment httpd2-pol2 --image=registry.redhat.io/rhel8/httpd-24
```

### 6.2 Test Communication to pol1 (Expected: Blocked)

Get pol1 Pod IPs:
```bash
oc get pods -o wide -n pol1
```

Sample output:
```
NAME                           READY   STATUS    RESTARTS   AGE    IP             NODE                  NOMINATED NODE   READINESS GATES
httpd1-pol1-776f659d5f-j82pv   1/1     Running   0          2m5s   10.131.0.232   worker-0.example.com   <none>           <none>
httpd2-pol1-67c7bff97b-q7g4p   1/1     Running   0          73s    10.131.0.233   worker-0.example.com   <none>           <none>
```

Try from pol2:
```bash
oc rsh httpd1-pol2-547ccfdb4b-fp5gs
sh-4.4$ curl 10.131.0.232:8080
^C
```

### 6.3 Label Namespace and List Pod Labels

```bash
oc label namespace pol2 project=pol2
oc get pods -n pol1 --show-labels
oc get pods -n pol2 --show-labels
```

Sample output:
```
NAMESPACE: pol1
httpd1-pol1-776f659d5f-j82pv   app=httpd1-pol1
httpd2-pol1-67c7bff97b-q7g4p   app=httpd2-pol1

NAMESPACE: pol2
httpd1-pol2-547ccfdb4b-fp5gs   app=httpd1-pol2
httpd2-pol2-7d55756789-gvq7c   app=httpd2-pol2
```

### 6.4 Create `allow-httpd1-pol2.yaml`

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-pod-and-namespace-both
spec:
  podSelector:
    matchLabels:
      app: httpd1-pol1
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            project: pol2
        podSelector:
          matchLabels:
            app: httpd1-pol2
      ports:
      - port: 8080
        protocol: TCP
```

### 6.5 Apply and Test

```bash
oc project pol1
oc apply -f allow-httpd1-pol2.yaml
oc project pol2
oc rsh httpd1-pol2-547ccfdb4b-fp5gs
sh-4.4$ curl 10.131.0.232:8080
<head>
Output Omitted
</head>
```

---

