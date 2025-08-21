# Network Policy

## Deny all Policy pods cannot communicate each other:

## 1. Create a deployment with nginx image in network namespace

```bash
kubectl create namespace network
```

## 2. Create declarative deployment:

```bash
kubectl create deployment nginx --image=nginx --replicas=4 -n network
```

## 3. Create yaml file of network policy with deny all policy with denyall.yaml file name:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: network
spec:
  podSelector: {}            # selects all Pods in default namespace
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []
```
## 4. Apply the yaml file
```bash
kubectl apply -f denyall.yaml 
```
## 5. Now go inside the node 1 pod and curl node 2 ip address in node 1 pod and u can see the you are not able to curl.

```bash
kubectl get pods -o wide -n network
```
## 6.Now go insdie the pod with the given command:

```bash 
kubectl exec -it <podname> -n network -- /bin/bash
```
## 7.Inside pod use  curl command with ip address

```bash
curl (node 2 ip addrres)
```
