# 02 - Cluster Hardening

# Accounts and Service Accounts

## Accounts

> **ServiceAccounts** are managed by Kubernetes API and are used by pods.
**Accounts** instead are represented by certificates and issued by the **Identity Management** of your provider. 

There are no *real users* in Kubernetes. A **User** in Kubernetes is just someone who holds a **Certificate** signed by the Kubernetes CA
> 

How to issue a new user:

- Create a `CertificateSigningRequest` object
- Kubernetes signs the certificate and updates the `csr`
- The certificate is now available to use (.crt)

## User creation in practice:

```bash
openssl genrsa -out tups.key 2048
openssl req -new -key tups.key -out tups.csr # only set Common Name = tups

# create CertificateSigningRequest with base64 tups.csr
# https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests
cat tups.csr | base64 -w 0
k apply -f tups.yaml

# Approve the csr
k certificate approve tups
k get csr tups -o yaml

# add new KUBECONFIG
k config set-credentials tups --client-key=tups.key --client-certificate=tups.crt
k config set-context tups --cluster=kubernetes --user=tups
k config view
k config get-contexts
k config use-context tups
```

Once a **Certificate** is issued it cannot be invalidated. If a **Certificate** is leaked the options are:

- Delete associated RBAC roles
- Block the username until certificate is expired
- Create a new **CA** and re-issue all user’s certificates.

## ServiceAccounts

> `ServiceAccounts` are *namespaced*. These can be used by pods to talk with Kubernetes API. A `default` service account is included in every namespace. They use a `Secret` with a token to authenticate against Kubernetes API. The token get created at service account creation.
> 

```yaml
...
spec:
	serviceAccount: account-name
...
```

Token gets mounted on the pod filesystem. Find them with `mount | grep token`. This happens automatically when a pod is configured to run with a `ServiceAccount`.

`ServiceAccount` can authenticate to the Kubernetes API via `curl` calls using the Bearer token:

```bash
# From inside a pod
curl https://kubernetes -k -H "Authorization: Bearer token-inside-secret"
```

When is not necessary that a `ServiceAccount` authenticate against Kubernetes API in a pod it can be disabled in the pod’s configuration:

```yaml
...
spec:
	serviceAccount: account-name
	automountServiceAccountToken: false
...
```

Depending on the `automountServiceAccountToken` value you will find or not the token mounted as a volume for the container.

## Tips:

- Create required `ServiceAccount` per application
- Restrict unnecessary access via RBAC

[Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

# RBAC

**RBAC** stands for *Role Based Access Control.*

Is a method of regulating access to resources based on the roles of individual users. In the `kube-apiserver` configuration is controlled by the `--authorization-mode` parameter.

> In RBAC you list only what you want to be **allowed.** Everything not specified will be denied (whitelist model).
> 

---

`Role` - Namespaced

`ClusterRole` - Cluster-wide

Set of permission:

- can edit pods
- can read secrets

`RoleBinding` - Namespaced

`ClusterRoleBinding` - Cluster-wide

Who get a set of permissions

- bind a `Role/ClusterRole` to something

---

Since the `ClusterRole` is a cluster-wide resourced it can also be combined with a `RoleBinding` granting the user the set of permission described to a singular namespace.

Indeed, `Roles` can’t be combined with `ClusterRoleBinding` since the first is a namespaced resource.

> Permission are additive. This results in the user getting the sum of the permission set granted.
> 

- Examine existing `RoleBindings` and `ClusterRoleBindings` to determine what permissions a `serviceAccount` has
- Design your `RBAC`setup in such a way that service accounts don't have unnecessary
permissions
- You can bind multiple roles to an account. Use this to keep `Roles` separate rather than
overloading them with a lot of permissions
- You can bind a `ClusterRole` with a `RoleBinding` to provide the necessary permissions only
within the `RoleBinding`'s namespace

## Important notes:

- You **can’t explicitly** **deny** a permission via RBAC
- You **can’t explicitly** **deny** to access a content resource via RBAC (es: list secret without content. having the verb `list` allows also to read the secret itself)

[Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

# Restricting Kubernetes API

> An HTTP request through API Server follows the *Authentication → Authorization → Admission Control* workflow.
> 

API Requests are always tied to:

- A normal user
- A `ServiceAccount`
- As anonymous requests

## **Security checklist**

- [ ]  **Don’t allow** anonymous access
- [ ]  Close **insecure** ports
- [ ]  **Don’t expose** ApiServer to the outside
- [ ]  Restrict access from nodes to API (`NodeRestriction`)
- [ ]  Prevent **unauthorized** access via RBAC
- [ ]  Prevent pods from accessing the API
- [ ]  Put the API Server behind a **firewall** or **allowed IP ranges**

## Anonymous Access

To enable anonymous access:

```bash
kube-apiserver \
... \
--anonymous-auth=true|false
```

Now HTTP REST calls can be made anonymously. This should be set to `true` because of some internal API calls Kubernetes makes under the hood.

In fact starting from Kubernetes 1.16+ it is `true` by default.

Calls are made with the `system:anonymous` user.

## Insecure Access

***This is not allowed anymore starting Kubernetes v1.20***

```bash
kube-apiserver \
...\
--insecure-port=8080
```

When this is **NOT** enabled, the workflow of an API request would be:

- `kubectl` make a request using a client cert
- Request is routed on HTTPS
- The `apiserver` uses a server certificate to authenticate the request.

Enabling Insecure access routes the requests via HTTP instead ad bypasses the authentication and authorization modules. This grants all rights also to `system:anonymous` user.

## Performing manual API requests

From the `config` file get the:

- `certificate-authority-data` as ***ca***
- `client-certificate-data` as ***crt***
- `client-key-data` as ***key***

Example call:

```bash
curl https://kubernetes-apiserver:6443 --cacert ca --cert crt --key key
```

## NodeRestriction AdmissionController

`NodeRestriction` serve to avoid a certain set of `labels` on nodes can be edited.

Enabling the Admission Controller:

```bash
kube-apiserver \
...\
--enable-admision-plugins=NodeRestriction
```

This limits the Node labels a `kubelet` can modify:

- It cannot edit other nodes’ labels
- It cannot edit other nodes’ pods labels
- It cannot edit some specific sets of label on itself `node-restriction.kubernetes.io/*`

Thus it can edit labels only on itself and its running pod except the excluded ones.

# Upgrading Kubernetes

Upgrade a master node:

```bash
# Upgrading only kubeadm first
apt update
apt-mark hold kubelet kubectl
apt-cache search kubeadm | grep 1.22
apt install kubeadm=1.22.5-00
apt-mark hold kubeadm

# Now upgrade the Kubernetes components
kubeadm upgrade plan
kubeadm upgrade apply v1.22.5

# Now upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt install kubelet=1.22.5-00 kubeadm=1.22.5-00
systemctl restart kubelet

# Node is upgraded!
```

Upgrade a worker node:

```bash
# Upgrading only kubeadm first
apt update
apt-mark hold kubelet
apt-cache search kubeadm | grep 1.22
apt install kubeadm=1.22.5-00
apt-mark hold kubeadm

# Now upgrade the Kubernetes components
kubeadm upgrade node

# Now upgrade kubelet
apt-mark unhold kubelet
apt install kubelet=1.22.5-00 
systemctl restart kubelet

# Node is upgraded!
```

---

- Use RBAC to control user permissions within the API. Make sure accounts do not have
permissions they do not need.
- Limit network access to the API to prevent attackers from being able to communicate
with it.
- Keep Kubernetes up to date with latest patches.

[Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)