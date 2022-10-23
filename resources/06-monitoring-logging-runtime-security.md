# 06 - Monitoring, Logging, Runtime Security

# Understanding Behavioral Analytics

- Behavioral analytics is the process of monitoring what is happening within a system to
detect malicious activity.
- One way to perform behavioral analytics in Kubernetes is to use tools like Falco.

# Analyzing Container Behavior with Falco

- You can run Falco from the command line with the `falco` command. View options with
`falco --help`.
- Use `-r` to pass in a Falco rules file.
- Use `falco --list` to see all available condition/output fields.
- Use `-M` to set the number of seconds Falco should collect data for

# Ensuring Containers are Immutable

- ***Immutability*** means that containers do **not** change at runtime by downloading and running
new code.
- Containers that use privileged mode (i.e., `securityContext.privileged: true`) may
also be considered *mutable*.
- Use `container.securityContext.readOnlyRootFilesystem` to prevent a container
from writing to its file system.
- If an application needs to write to files, such as for caching or logging, you can use an
`emptyDir` volume alongside `readOnlyRootFilesystem`.
- Remove shells, make filesystem readonly, run as **non** *root* user
- Override default command in Kubernetes (poor solution)
- Use `StartupProbe` to make filesystem read only for example (poor solution). This behaves like a `livenessProbe`
- Enforce read-only filesystem via `SecurityContexts` and `PodSecurityPolicies` (preferred way) then use an `EmptyDir` volume in case the application needs some writable location.

# Auditing

## Understanding Audit Logs

- Kubernetes auditing allows you to capture logs of all changes made through the
Kubernetes API.
- Auditing in Kubernetes has four kinds:
    - `RequestReceived` - The stage for events generated as soon as the audit handler receives the request, and before it is delegated down the handler chain.
    - `ResponseStarted` - Once the response headers are sent, but before the response body is sent. This stage is only generated for long-running requests (e.g. watch).
    - `ResponseComplete` - The response body has been completed and no more bytes will be sent.
    - `Panic` - Events generated when a panic occurred.
- In audit log policy rules, resources identifies the Kubernetes resource types the rule
applies to.
- In audit log policy rules, namespaces limits the rule to only specific namespaces.
- In audit log policy rules, level identifies how detailed the log data should be for the rule:
    - `None` - don't log events that match this rule.
    - `Metadata` - log request metadata (requesting user, timestamp, resource, verb, etc.) but not request or response body.
    - `Request` - log event metadata and request body but not response body. This does not apply for non-resource requests.
    - `RequestResponse` - log event metadata, request and response bodies. This does not apply for non-resource requests.

## Setting up Audit Logging

Create an Audit policy:

```yaml
apiVersion: audit.k8s.io/v1 
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]
```

- Define audit rules in the audit policy configuration file. Rules are evaluated in sequential order.
- API Server flags for audit logging:
    - `--audit-policy-file` - Points to the audit policy config file
    - `—-audit-log-path` - The location of log output files
    - `--audit-log-maxage` - The number of days to keep old log files
    - `--audit-log-maxbackup` - The number of old log files to keep.

Both audit policy file and audit logs folder must be mounted inside the API Sever specs:

```yaml
# add new Volumes
volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy/policy.yaml
      type: File
  - name: audit-logs
    hostPath:
      path: /etc/kubernetes/audit-logs
      type: DirectoryOrCreate

# add new VolumeMounts
volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy/policy.yaml
    name: audit-policy
    readOnly: true
  - mountPath: /etc/kubernetes/audit-logs
    name: audit-logs
    readOnly: false
```

Refer to the docs to know how to properly configure this:

[Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)