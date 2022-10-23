# Useful Commands

Create `Roles`: 

```bash
kubectl create role role-name --verb= --resource= --namespace=

# Examples
# Create a Role named "pod-reader" that allows users to perform get, watch and list on pods:
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods

# Create a Role named "pod-reader" with resourceNames specified:
kubectl create role pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod

# Create a Role named "foo" with apiGroups specified:
kubectl create role foo --verb=get,list,watch --resource=replicasets.apps

# Create a Role named "foo" with subresource permissions:
kubectl create role foo --verb=get,list,watch --resource=pods,pods/status

# Create a Role named "my-component-lease-holder" with permissions to get/update a resource with a specific name:
kubectl create role my-component-lease-holder --verb=get,list,watch,update --resource=lease --resource-name=my-component
```

Create `RoleBindings`:

```bash
kubectl create rolebinding rolebinding-name --role= --namespace= --serviceaccount/--user

# Examples
# Within the namespace "acme", grant the permissions in the "admin" ClusterRole to a user named "bob":
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme

# Within the namespace "acme", grant the permissions in the "view" ClusterRole to the service account in the namespace "acme" named "myapp":
kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme

# Within the namespace "acme", grant the permissions in the "view" ClusterRole to a service account in the namespace "myappnamespace" named "myapp":
kubectl create rolebinding myappnamespace-myapp-view-binding --clusterrole=view --serviceaccount=myappnamespace:myapp --namespace=acme
```

Create `ClusterRoles`:

```bash
# Create a ClusterRole named "pod-reader" that allows user to perform get, watch and list on pods:
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods

# Create a ClusterRole named "pod-reader" with resourceNames specified:
kubectl create clusterrole pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod

# Create a ClusterRole named "foo" with apiGroups specified:
kubectl create clusterrole foo --verb=get,list,watch --resource=replicasets.apps

# Create a ClusterRole named "foo" with subresource permissions:
kubectl create clusterrole foo --verb=get,list,watch --resource=pods,pods/status

# Create a ClusterRole named "foo" with nonResourceURL specified:
kubectl create clusterrole "foo" --verb=get --non-resource-url=/logs/*

# Create a ClusterRole named "monitoring" with an aggregationRule specified:
kubectl create clusterrole monitoring --aggregation-rule="rbac.example.com/aggregate-to-monitoring=true"
```

Create `ClusterRoleBindings`:

```bash
# Across the entire cluster, grant the permissions in the "cluster-admin" ClusterRole to a user named "root":
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root

# Across the entire cluster, grant the permissions in the "system:node-proxier" ClusterRole to a user named "system:kube-proxy":
kubectl create clusterrolebinding kube-proxy-binding --clusterrole=system:node-proxier --user=system:kube-proxy

# Across the entire cluster, grant the permissions in the "view" ClusterRole to a service account named "myapp" in the namespace "acme":
kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

Verify permission for a user:

```bash
k auth can-i verb resource -n namespace --as system:serviceaccount:namespace:sa-name
```

Trivy:

```bash
trivy image -s HIGH,CRITICAL image:tag
```

Expose a pod:

```bash
# ClusterIP
k expose pod pod-name --port 80 --name service-name

# Test communication to another pod
k exec pod-name2 -- curl pod-name-service

# Load Balancer
k expose deployment hello-world --type=LoadBalancer --name=my-service
```

Secrets:

```bash
# Help for creating secrets by given type
k create secret tls|generic -h
```