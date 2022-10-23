# 01 - Cluster Setup

# NetworkPolicies

- Firewall rules in Kubernetes
- Implemented by CNI (Calico/Wave)
- Namespaced resources
- Multiple rules are regolated by an `OR` condition
- Multiple `NetworkPolicies` applied to the same pods are regolated by `Union` rule (merged)
- Default deny for all + targeted policies to block all but necessary traffic in a namespace
- An “empty” `NetworkPolicy` is a `default-deny`
- Use an empty `podSelector: {}` to apply the NP to all pods in a given namespace
- Pay attention to the difference between *one rule with multiple selectors* and *multiple rules*

[Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

# GUI Elements

- `kubectl proxy` create a proxy server in http to the API Server from the local machine for REST requests
- `kubectl port-forward` forwards from [localhost](http://localhost) to a pod in http or tcp
- Avoid exposing unnecessary services

# CIS Benchmarks

A set of standard and best practices for a secure Kubernetes cluster

- `kube-bench` runs automated tests to check how well the cluster conforms to the CIS Benchmark
- `kubeadm` clusters use a config file for `kubelet` at `/var/lib/kubelet/config.yaml`
- In case of changes of control plane components all manifests are at `/etc/kubernetes/manifests`
- Run `kube-bench` to have a list of issues and remediations for your cluster

```
sha512sum filename

# Get the downloaded hash into a file.
# Write the vendor provided hash into the same file and compare them with uniq
echo $HASH_ONE > comparison
echo $HASH_TWO >> comparison

cat comparison | uniq
```

[Martin White - Consistent Security Controls through CIS Benchmarks](https://www.youtube.com/watch?v=53-v3stlnCo)

# Ingress

- Implement TLS termination
- Store TLS in a secret and pass it via `spec.tls[].secretName`
- The `Ingress` object is just a wrapper for the nginx (or ingress-controller specific) config
- An `host` key can map multiple paths
- Verify the CN of given TLS certificate

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

[Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

# Node Metadata

This applies only for clusters on Cloud Providers such as AWS, GCP, Azure. Not included in CKS exam curriculum

- Metadata service API is reachable from VMs
- Contains cloud credentials for VMs
- Can contain provisioning data
- Access can be restricted via `NetworkPolicy`

# Verify Platform Binaries

- Compare the HASH/SHA value provided by the source with the downloaded file

```bash
sha512sum filename

# Get the downloaded hash into a file.
# Write the vendor provided hash into the same file and compare them with uniq
echo $HASH_ONE > comparison
echo $HASH_TWO >> comparison

cat comparison | uniq
```

# General

- Kubernetes uses known ports. Protect them
- Secure also GUIs access
- Validate binaries. Do not execute compromised software