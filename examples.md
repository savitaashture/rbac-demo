### AUTHENTICATION:

```
openssl genrsa -out sa.key 2048
openssl req -new -key sa.key -out sa.csr -subj "/CN=sa/O=dev"
openssl x509 -req -in sa.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out sa.crt -days 500
```
Verify
```
openssl x509 -in "sa.crt" -text -noout
```

#kubectl config set-cluster sa --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --server https://10.180.1.94:6443
### kubectl adds user info to kubeconfig
```
kubectl config set-credentials sa --client-certificate=sa.crt --client-key=sa.key --embed-certs=true

kubectl config set-context sa --cluster=kubernetes --user=sa
kubectl config get-contexts
kubectl config use-context sa
```

### AUTHORIZATION
#### Role
```
cat <<EOF | kubectl create -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: demo
  name: demorole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: demorb
  namespace: demo
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: demopod
  apiGroup: rbac.authorization.k8s.io
EOF
```
### Verification
```
kubectl get pods -n demo ---- works
kubectl delete pod test -n demo ---- doesnot work (because it has access to just list and get operations)
```
### Access to all operation at demo namespace
```
cat <<EOF | kubectl create -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: demo
  name: demopod
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: demorb
  namespace: demo
subjects:
- kind: User
  name: sa
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: demopod
  apiGroup: rbac.authorization.k8s.io
EOF
```

#### CLUSTERROLE
```
cat <<EOF | kubectl create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: democlusterrole
rules:
- apiGroups: ["*"]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: demorb
subjects:
- kind: User
  name: sa
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: demopod
  apiGroup: rbac.authorization.k8s.io
EOF

cat <<EOF | kubectl create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: demopod
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
EOF

cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: demorb
subjects:
- kind: User
  name: sa
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: demopod
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Verification 
1. First clusterrole will allow get and list operations for pods for all the namespaces.
2. Second clusterrole will allo all the operations at the cluster levels.
