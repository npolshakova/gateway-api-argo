# gateway-api-argo

### Requirements
1. Setup argo kubectl plugin

1. Setup k3s cluster

./setup-env.sh

2. Install argo 

```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

3. Pick a gateway! 

### Envoy Gateway

```
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.1.2 -n envoy-gateway-system --create-namespace

```


```
kubectl apply -f gateways/envoy/gateway.yaml 
```

### Istio Ambient

```
istioctl install --set profile=ambient
```

```
kubectl apply -f gateways/istio-ambient/gateway.yaml
```

4. Allow Argo Rollouts to edit Http Routes

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-controller-role
  namespace: argo-rollouts
rules:
  - apiGroups:
      - gateway.networking.k8s.io
    resources:
      - httproutes
    verbs:
      - get
      - patch
      - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gateway-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gateway-controller-role
subjects:
  - namespace: argo-rollouts
    kind: ServiceAccount
    name: argo-rollouts

```

5. Create an HTTPRoute

```
---
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: argo-rollouts-http-route
  namespace: default
spec:
  parentRefs:
    - name: eg
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: argo-rollouts-stable-service
      kind: Service
      port: 80
    - name: argo-rollouts-canary-service
      kind: Service
      port: 80

```

6. Perform a Canary

```
kubectl argo rollouts get rollout rollouts-demo
```

```
kubectl argo rollouts promote rollouts-demo
```

7. View in Argo Rollouts UI

```
kubectl port-forward -n <gw-ns> port-forward service/<gw-svc> 8888:80 &
```
