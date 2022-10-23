# 04 - Minimizing Microservice Vulnerabilities

# Hacking secrets in ETCD

`Secrets` in ETCD are not encrypted, they are stored in plain text.

```bash
# access secret int etcd
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
ETCDCTL_API=3 etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt endpoint health

ETCDCTL_API=3 etcdctl --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key --cacert /etc/kubernetes/pki/etcd/ca.crt get /registry/secrets/mynamespace/secret
# Search keys with /registry/secrets/namespace/secretname

```

## Encrypting secrets in ETCD at rest

*Create an `EncryptionConfiguration` file and add it to `kube-apiserver` parameters*

In both cases create a base64 password to set as encryption key (`secret:` key)

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - identity: {}
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - aescbc:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2

# This configuration means that the secrets will be stored in plain text since the first provider
# is identity: {}, but secrets encrypted with aesgcm and aescbc protocls
# can be read by the apiserver.
```

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aesgcm:
          keys:
            - name: key1
              secret: c2VjcmV0IGlzIHNlY3VyZQ==
            - name: key2
              secret: dGhpcyBpcyBwYXNzd29yZA==
      - identity: {}

# This configuration means that the secrets will be stored with aesgcm encryption since it's the
# first provider, but the apiserver will be able to read also unencrypted secrets
# since it is specified as second provider.
# If that wasn't set apiserver would be able to operate only with aesgcm encrypted secret
# without being able to read plain text ones.
# Having the identity provider at the end is mandatory.
```

```bash
kube-apiserver \
... \
--encryption-provider-config=path/to/encryptionconfiguration

# Create also the VolumeMount and Volume HostPath definition for the file.
```

> To apply the encrypton to *all already existing secrets* do the following after the `EncryptionConfiguration` is set:
> 

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
# Self substitution
```

> To de-crypt  *all already existing secrets* do the following after the `EncryptionConfiguration` is set with the first provider as `identity: {}`:
> 

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
# Self substitution
```

> The above command replaces secrets with themselves following the specification of the applied `EncryptionConfiguration`
> 

---

Encrypting secrets in ETCD means that when r*eading the secret from ETCD directly* you cannot access the value since is **masked**. When *reading the secret via API Server* though the secret is still available in **base64 format**, so it's **easy to decode.**

The `IdentityProviders` are used in sequence

[Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

---

# OS Level security domains

## Security context

- `securityContext` offers a variety of security and access control-related settings.
- `spec.securityContext` sets securityContext settings at the Pod level. These
settings apply to all containers in the Pod.
- `spec.containers[ ].securityContext` sets securityContext settings at the
container level. These settings apply to individual containers within the Pod.

***Security Contexts***  allows to define for example *UserID* or *GroupID*, *Privileged* or *Unprivileged* access. They work on different levels (`Deployment` level, `container` level)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
	# This is at pod-level. Every container in the pod will be affected by this.
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
...
```

Forcing a container to run as `NonRoot`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
		# This is at pod-level. Every container in the pod will be affected by this.
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
...
	containers:
	- image: busybox
		...
		# This is at container level. The container is now not allowed to run as root.
		securityContext:
			runAsNonRoot: true

# This example is a paradox since at pod level we defined the pod to run as root,
# but at container level we specify to not run as root.
# This can be used only if the container itself it's not made to run as root.
# For example busybox comes running with a root access
```

[Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

## Privileged containers

> By default ***Docker*** runs container as **unprivileged.** **Privileged**  access should be given in case of need to access device resource or for nested Docker (Docker runtime running inside a container)
> 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
...
	containers:
	- image: busybox
		...
		# This is at container level. The container is now allowed to run as root.
		securityContext:
			privileged: true

# This example shows the difference of running root inside a container and running 
# as root from the runtime.
#
# By default the container runtime runs as unprivieged (see quote above).
# This means that a container running with a root user such as BusyBox can use
# root permission only inside itself, but not on the host system.
#
# When it is run with the --privileged flag by Docker or with the above specs
# in Kubernetes instead, the container is able to perform root operations also 
# on the host system, then outside its container filesystem.
#
# This is because a 1:1 mapping between container user and host user happens.
```

| Privileged | PrivilegeEscalation |
| --- | --- |
| Means that container user 0 (root) is directly mapped to host user 0 (root) | Controls wheter a process can gain more privileges than its parent process |

Allowing `PrivilegeEscalation`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
...
	containers:
	- image: busybox
		...
		# This is at container level. The container is now allowed to run as root.
		securityContext:
			allowPrivilegeEscalation: true|false

# Based on the value of allowPrivilegeEscalation we can check its behavior with:
# cat /proc/PID/status
#
# When set to true the NoNewPrivs is set to 0 meaning that new privileges were
# not given since there was no escalation need and the container is already
# allowed to PrivilgeEscalation
#
# When set to false instead we will see the number of NoNewPrivs requested since
# the container is trying to escalate privileges
```

## Pod Security Policies

> These are **cluster-level** resources. Since it is an AdmissionController every pod must conform from a security perspective to these policies. This controller must be enabled on the API Server before using it:
> 

```bash
kube-apiserver \
--enable-admission-plugins=PodSecurityPolicy
```

Once activated a pod must be satisfy at least one policy. So activation of admission controller must be done after creating any policies.

- Use Pod security policies to enforce desired security configurations for new Pods.
- Pod security policies can reject Pods that don't meet the desired standard, or modify
Pods by applying default settings.

`PodSecurityPolicy` can allow/deny privileged mode, host namespaces, volumes access, limit hostPath and so on.

Example of a `PodSecurityPolicy`:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  allowPrivilegeEscalation: false
	privilege: false
	...

# This policy will prevent pods to be created cluster wide
# if they have set allowPrivilegeEscalation or privilege
```

> Rule of thumb: always create the `PodSecurityPolicy` **BEFORE** enabling them on the API Server otherwise no pods will be able to be created.
> 

> It's important to note that if Deployment for example is using a `ServiceAccount` to run which **can't** use the `PodSecurityPolicy` created the containers won't run since they ***will not satisfy*** the policy.
> 

For a user to use pod security policies it must be authorized to use the policy via **RBAC.**

The `use` verb in a `Role` or `ClusterRole` allows users to use `PodSecurityPolicy`.  Every new pod in order to be scheduled must use an account which is authorized to use and satisfy at least one policy.

```yaml
# Make default:default user allowed to use PodSecurityPolicy
k create role psp-access --verb=use --resource=podsecuritypolicies
k create rolebinding psp-access --role=psp-access --service-account=default:default

# Now default:default can use the PodSecurityPolicy, thus resources
# configured to run with that ServiceAccount are allowed to run.
#
# Now that the ServiceAccount can use the policy, obviously every pod/container must
# be compliant to run.
```

Policy created by a User deploying resources:

- The user creating the Pod has access to use the policy.
- Control which users can create Pods according to which policies.
- Does not work well for `Pods` that are not created directly by users (think `Deployments`, `ReplicaSets`, `DaemonSets`, etc.). Since it's the User to have rights on the policy when the replication controller creates a Deployment the pods n it won't be able to use the `PodSecurityPolicy`. Only pods manually created by that user will have the policy attached.

Policy created for `ServiceAccount` (preferred way):

- The pod's service account has access to use the policy
- Works with indirectly-created `Pods`, `Deployments`, etc since resources are not created by a User and they use a `ServiceAccount` in the specification

**To sum up:**

- To use Pod security policies, you must first enable the PodSecurityPolicy admission
controller. Use the `--enable-admission-plugins=PodSecurityPolicy` flag on kube-apiserver to do this.
- In order to create a Pod, a user (or the Pod's `ServiceAccount`) must be authorized to use
a `PodSecurityPolicy` via the `use` verb in RBAC.
- To apply a `PodSecurityPolicy` within the context of a specific namespace, authorize a
`ServiceAccount` in that namespace to use the policy.
- The `PodSecurityPolicy` must be explicitly satisfied in the resource's specs

[Pod Security Policies](https://kubernetes.io/docs/concepts/security/pod-security-policy/)

# Using OPA Gatekeeper

- Open Policy Agent (OPA) Gatekeeper allows you to enforce custom policies on any k8s
object at creation time.
- `ConstraintTemplate` define reusable constraint logic and any parameters that can be
passed in.
- `Constraint` objects apply a `ConstraintTemplate` to a specific group of potential incoming
objects, alongside specific parameters.
- The `Constraint` is an implementation of a `ConstraintTemplate`
- It works as an `AdmissionController` to allow/deny new workloads according to regulatory policies deployed cluster-wide
- Ensure that no other `admission-control-plugins` are listed in the API Server manifest before installing OPA Gatekeeper
- Policies are not retroactive so they won't affect previously denied resources (for example)

## Using Policies

Create a `ContraintTemplate`:

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8salwaysdeny
spec:
  crd:
    spec:
      names:
        kind: K8sAlwaysDeny
      validation:
        openAPIV3Schema:
          properties:
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8salwaysdeny
        violation[{"msg": msg}] {
          1 > 0
          msg := input.parameters.message

#
# 1>0 is the condition for violation.
# This means that when a Constraint will be created based on this template,
# a violation will always be triggered by the creation of a specified resource.
# See below.
#

# OPA will create a CustomResource named after the kind: key.
# Then a Constraint of that kind can be created:
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAlwaysDeny
metadata:
  name: pod-always-deny
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    message: "ACCESS DENIED!"

#
# This Constraint is applied to all pod kind resource.
# Since the ContraintTemplate which this Constraint is based has as rule
# 1>0, on the creation of every new pod the violation is triggered.
#
```

> As a *best practice* set the error message in the `Constraint` instead of `ConstraintTemplate` to upgrade the messages dynamically.
> 

> **Violations** inside the cluster can be checked for correction by inspecting the `Constraint:`
> 

```bash
k describe K8sAlwaysDeny pod-always-deny
```

[The Rego Playground](https://play.openpolicyagent.org)

[GitHub - BouweCeunen/gatekeeper-policies: Kubernetes admission control on Kubernetes with Gatekeeper policies.](https://github.com/BouweCeunen/gatekeeper-policies)

[Policing Your Kubernetes Clusters with Open Policy Agent (OPA) - Mark Puddick & Amith Nambiar](https://www.youtube.com/watch?v=RDWndems-sk)

---

# Managing Kubernetes Secrets

- Secrets store sensitive data, and can pass it to containers.
- You can pass secret data to a container using either environment variables or mounted
volumes.
- To retrieve secret data from the command line, you can use `kubectl get -o yaml` to
get the base64-encoded data, then decode it with `base64 -decode`

# Container Runtime Sandboxes

> *Containers are not contained. Just because it runs in a container doesn't mean is protected.*
> 

**Sandbox** place themselves between the container and the host kernel to intercept `SYSCALLS`.

The ***Container Runtime interface*** (CRI) allows the `kubelet` to use different runtimes instead of being strictly tied to *Docker.*

- ***Container Runtime Sandboxes*** provide a specialized runtime with additional layers of
isolation, allowing you to run untrusted workloads more securely.
- `gVisor` creates a runtime sandbox by running a Linux application kernel within the host OS.
`runsc` is the 0CI-compliant container runtime that allows Kubernetes to interface with `gVisor`. It runs a simulated Linux Kernel into a real one.
- Kata Containers create a sandbox by transparently running containers inside of
lightweight virtual machines.
- Use a `RuntimeClass` to define a specialized container runtime configuration, such as one
that will use `gVisor/runsc`.
- Set the `runtimeClassName` property in a Pod specification to make the Pod use the
container runtime sandbox.

| KataContainer | gVisor |
| --- | --- |
| Stronger separation layers | Another layer of separation |
| Hypervisor/VM based. Every container runs in a private VM | NOT hypervisor/VM based, uses runsc |
| Uses QEMU as default (needs virtualisation like nested virtualisation in cloud) | Simulates kernel SYSCALLS with limitations. gVisor accept the calls made by the process and translates/limitates them for the host kernel |

## Using a RuntimeClass

Create the `RuntimeClass`:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor 
handler: runsc
```

Once created reference it in a pod:

```yaml
...
spec:
	runtimeClassName: gvisor
...
```

To check if the container is using the correct runtime a simple `uname -r` is enough to determine that the runtime kernel version is different from the host's kernel. This is because the pod using the sandbox is running on a different container by design.

Also `dmesg` is useful in this case.

[OCI, CRI, ??: Making Sense of the Container Runtime Landscape in Kubernetes - Phil Estes, IBM](https://www.youtube.com/watch?v=RyXL1zOa8Bw)

[Sandboxing your containers with gVisor (Cloud Next '18)](https://www.youtube.com/watch?v=kxUZ4lVFuVo)

[Kata Containers An introduction and overview](https://www.youtube.com/watch?v=4gmLXyMeYWI)

[Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/)

---

# Understanding pod-to-pod mTLS

- Mutual Transport Layer Security (mTLS) means clients and servers mutually authenticate
with each other and encrypt their communications. It's bilateral.
- You can obtain certificates using the Kubernetes API.

**Certificates** should be rotated often.

Injection happens via `Proxy`/`SideCar` container. In this way is the proxy container that actually make communication happens between pods ***not*** the pods themselves.
This `SideCar` container is often deployed by an external manager like a **Service Mesh** service.

[A Kubernetes engineer's guide to mTLS](https://buoyant.io/mtls-guide/)

---

# Signing Certificates

- Create a CertificateSigningRequest object to request a new certificate.
- Manage, approve, or deny requests via the command line with kubectl certificate.
- Once approved, the signed certificate can be retrieved from the status. certificate
field of the CertificateSigningRequest.

---

- Secrets can be read also via `docker/crictl inspect container-id`
- `inspect` output also gives the `PID` on the host machine. With that you can inspect the filesystem with `ls /proc/PID`
- Securing node access allows to secure secrets mounted on the container