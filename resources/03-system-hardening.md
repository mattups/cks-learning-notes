# 03 - System Hardening

# Host namespace

- Protect your hosts from attacks that might come from within containers.
- Beware of Pod settings like `hostPID, hostIPC,` and `hostNetwork`. Use them only when
absolutely necessary!
- Beware of using privileged mode for containers with `securityContext.privileged`
Use this only when absolutely necessary!

# IAM Roles

- If you are running Kubernetes in AWS, your container applications may be able to access
AM credentials.
- Avoid providing unnecessary permissions to IAM roles (principle of least privilege).
- If your app does not need IAM access, consider blocking access to IAM credentials via
firewall, NetworkPolicy, etc.

# Network-level security

- By default, anyone who can access the cluster network can communicate with all Pods
and Services in the cluster.
- When possible, limit access to the cluster network from outside.

# AppArmor

> **AppArmor** is a Linux kernel security module that allows granular control over what
individual programs can and cannot do.
> 

Profiles can be loaded in different modes:

- Load a profile in **enforce** mode (sometimes called "enforcing the profile") to actively
prevent programs from doing anything the profile does not allow.
- Load a profile in **complain** mode to simply report on what the program is doing.
- Use the `apparmor_parser` command to load an AppArmor profile from a file. It will load
the profile in *enforcing* mode by default.
- Use Pod annotations to apply an AppArmor profile to a container. For example:
`container.apparmor.security.beta.kubernetes.io/nginx: localhost/k8s-deny-
write`

***AppArmor profiles must be present and loaded on every node of the cluster.***

To edit the name of an AppArmor profile edit the line:

```bash
profile profile_name flags=(attach_disconnected,mediate_deleted) {
```

Load it with `apparmor_parser profile` and verify it via `apparmor_status | grep profile`

## Profile types

- ***Unconfined:*** Process can escape
- ***Complain***: Process can escape but will be logged
- ***Enforce**:* Process cannot escape

## Useful commands

```yaml
# Show profiles
aa-status

# Generate profile
aa-genprof

# Put profile into complain mode
aa-complain

# Put profile into enforce mode
aa-enforce

# Update profile if app produced more usage logs. This will check syslog
# to analyze the behaviour and ask if allow what did in the logs.
aa-logprof

```

---

# SecComp

> Also known as “**Secure Computing mode**” it's a security facility in the Linux Kernel which restricts execution of syscalls. If a syscall is set to be blocked, SecComp will send a SIGKILL signal to it.
> 

**SecComp** uses profiles.

## Use in Kubernetes

- Create a `default.json` profile in `/var/lib/kubelet/seccomp`
- Configure the spec as follow:

```yaml
...
securityContext:
	seccompProfile:
		type: Localhost
		localhostProfile: profiles/default.json
...
```