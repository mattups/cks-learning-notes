# 05 - Supply Chain Security

# Minimizing Base Image Attack Surface

- Try to use images that run up-to-date software to minimize software vulnerabilities.
- Minimize the presence of unnecessary software in images that could increase security
risks.
- Beware of the possibility of images that have been compromised by an attacker.

# Whitelisting Allowed Image Registries

- Limit users to only trusted image registries to prevent them from running images from
untrusted sources in the cluster.
- You can limit registries using OPA Gatekeeper.

# Validating Signed Images

- Container images can be signed with a hash generated from the image contents.
- To validate the image, you can append the hash to the image reference in your container
spec with image: `imageName:tag@sha256:hash`.

# Image footprint and hardening

- To avoid running the container as the root user, make sure that the final `USER` directive in
the Dockerfile is **not** set to *root* or *0*. Create a specific user with the `USER` directive.
- Avoid using the `:latest` tag in the `FROM` directive.
- Try to avoid including unnecessary software in the final image.
- Avoid storing sensitive data such as passwords in the Dockerfile (for example, using the
`ENV` directive). Use Kubernetes `Secrets` instead.
- Minimize layers.
- Using multi-stage build reduces the image footprint resulting in smaller images.
- Make filesystem readonly with a `RUN chmod a-w /etc` for example beside using the correct `SecurityContext` in Kubernetes.
- Remove shell access with a `RUN rm -rf /bin/*` so unwanted binaries are not taken into the image.

# Analyzing Resource YAML Files

- When possible, avoid the use of host namespaces in your Pod configurations (i.e., with
`hostNetwork: true`, `hostIPC: true`, or `hostPID: true`).
- When possible, avoid using privileged containers with `securityContext.privileged: true`.
- Avoid running as user root or 0 in `securityContext.runAsUser`.
- Don't use the `:latest` tag, but instead use a specific, fixed tag to avoid downloading a
new and potentially unvetted image.

# Scanning Images for Known Vulnerabilities

- *Vulnerability scanning* allows you to scan images to detect security vulnerabilities that
have already been discovered and documented by security researchers.
- Trivy is a command-line tool that allows you to scan images by name and tag.
- Scan an image name and tag with Trivy like so: `trivy image nginx:1.14.1`
- In some cases, you may need to omit "image" like so: `trivy nginx:1.14.1`

# Scanning Images with an Admission Controller

- Admission controllers intercept requests to the Kubernetes API before objects are
created. They can allow objects to be created, prevent their creation, or make changes to
objects before creating them.
- The `ImagePolicyWebhook` admission controller allows you to use customisable logic to
approve or deny the creation of workloads based upon the container image being used.
- You can use the `ImagePolicyWebhook` admission controller to have an external application
scan images for vulnerabilities automatically as workloads are created.

# Setting Up an Image Scanner

- The `ImagePolicyWebhook` admission controller sends a JSON request to an external
service to determine if images are allowed.
- The external service provides a JSON response indicating whether the images are
allowed or disallowed.

# Configuring the ImagePolicyWebhook Admission Controller

- Use `--enable-admission-plugins` in the kube-apiserver manifest to enable the
`ImagePolicyWebhook` admission controller.
- Use `--admission-control-config-file` to specify the location of the admission
control configuration file.
- If the config files are on the host file system, you may need to mount them to the kube-
apiserver container.
- In the admission control config, `kubeConfigFile` specifies the location of a kubeconfig
file. This file tells `ImagePolicyWebhook` how to reach the webhook backend.
- In the admission control config, `defaultAllow` controls whether or not workloads will be
allowed if the backend webhook is unreachable.
- `server` fields accepts only `https` endpoints

# Behavioural Analytics at host and container level

## SysCalls

> **SysCalls** are the interface between the ***User Space***  and ***Kernel Space.*** Every process in the *User Space* creates via the process itself or a library (es: **glibc**) a *SysCall* to the **SysCall Interface** which will be managed by the **Kernel** in the *Kernel Space.*
> 

Tools like **SecComp**  and **AppArmor** places right before the *SysCall Interface* in order to intercept these and report if anything malicious is happening or detect anomalous calls.

### Strace

> **sTrace** is a tool that intercepts and logs system calls made by a process. Logs also signal and is good for tracing and debugging the syscall made by a process.
> 

```bash
strace ls /
# This will give you the full list of SYSCALLS made

strace -cw ls / 
# Output is smaller, you get syscalls in a table. Count and Summarize
```

> The `/proc` directory contains informations and connections to process and kernel. This is useful to study how a process works.
> 

```bash
# List all ETCD SYSCALLS
strace -p ETCD_PID

# List open file
cd /proc/ETCD_PID
# This gets the list of open sockets and files.
cd ft

# Read secret values from open files
k create secret generic usersecret --from-literal=user=tups
cat ID_DB_ETCD | grep 1234
# This would show in real time that in the DB there is that secret (unencrypted)
```

> *Secrets* and *Environment* *Variables* **can** be read from **anyone** who can access the `/proc`directory on the host.
> 

---

**Summing up:**

- Install a scanning webhook like trivy
- Create a `AdmissionConfiguration` (in an `admission-control.conf` file for example)
- The `AdmissionConfiguration` must point to a `.kubeconfig`
- In the `.kubeconfig` configure the `server`  and certificates. `server` must be https
- Edit the `kube-apiserver` config with `--enable-admission-plugins=ImagePolicyWebhook` and `--admission-control-config-file=path/to/admission-control.conf`
- Setup `VolumeMount` for the config file and add the `Volume`
- The file `admission-control.conf` must point to the `kubeconf`

[Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)