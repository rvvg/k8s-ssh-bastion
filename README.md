# k8s-ssh-bastion

A lightweight, secure SSH bastion for Kubernetes, designed to provide SSH access to your cluster network.

## Features

-   **Standard Base Image**: Uses the official `alpine:3.22` image.
-   **Runtime Installation**: Installs `openssh-server`, `bash`, and `shadow` at pod startup, ensuring latest security updates.
-   **Load Balancer Support**: Can automatically create an additional `LoadBalancer` service or configure the main service as one.
-   **Security**: Supports restricting access by IP ranges and disabling password authentication.
-   **High Availability**: Configurable PodDisruptionBudget, replicas, and TopologySpreadConstraints.
-   **Customizable**: Supports custom init scripts and full `sshd_config` overrides.

## Prerequisites

-   Kubernetes 1.19+
-   Helm 3.0+

## Installation

Add the repository:

```bash
helm repo add k8s-ssh-bastion https://rvvg.github.io/k8s-ssh-bastion/
helm repo update
```

Install the chart:

```bash
helm install bastion k8s-ssh-bastion/k8s-ssh-bastion
```

## Configuration

### Users

Add users and their public keys in `values.yaml`. Users are created at startup and keys are placed in `~/.ssh/authorized_keys`.

```yaml
users:
  alice: "ssh-rsa AAAAB3NzaC1yc2E..."
  bob: |
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
    ssh-rsa AAAAB3NzaC1yc2E...
```

### IP Whitelisting

Restrict SSH access to specific IP ranges:

```yaml
allowedIPRanges:
  - 1.2.3.4/32
  - 10.0.0.0/8
```

### Load Balancer

You can configure the main service as a `LoadBalancer` or create an additional one:

```yaml
service:
  type: LoadBalancer
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
```

Or keep `NodePort` for the main service and create a secondary `LoadBalancer`:

```yaml
service:
  type: NodePort
  createLoadBalancer: true
  loadBalancer:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

### Init Scripts

Run custom commands before starting `sshd`:

```yaml
initscripts:
  install-tools.sh: |
    #!/bin/bash
    apk add --no-cache curl jq
```

### Host Keys

By default, Helm will generate SSH host keys (RSA, ECDSA, Ed25519) during installation and store them in a Secret. These keys are reused across upgrades if the Secret exists.

If you want to provide your own keys (e.g., to prevent "Host key verification failed" warnings when migrating), you can specify them in `values.yaml`:

```yaml
hostKeys:
  rsa: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
  ecdsa: |
    -----BEGIN EC PRIVATE KEY-----
    ...
  ed25519: |
    -----BEGIN PRIVATE KEY-----
    ...
```

### ExternalDNS

To automatically configure DNS for your bastion service using [ExternalDNS](https://github.com/kubernetes-sigs/external-dns), add the annotation:

```yaml
service:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: bastion.example.com
```

### Values Reference

| Key | Description | Default |
|-----|-------------|---------|
| `image.repository` | Image repository | `alpine` |
| `image.tag` | Image tag | `3.22` |
| `replicaCount` | Number of replicas | `1` |
| `service.type` | Service type (`NodePort`, `LoadBalancer`) | `NodePort` |
| `service.nodePort` | Node port (if type is NodePort) | `30022` |
| `service.createLoadBalancer` | Create an additional LoadBalancer service | `true` |
| `service.annotations` | Annotations for the service | `{}` |
| `allowedIPRanges` | List of CIDRs allowed to connect | `[]` |
| `users` | Map of usernames to public keys | `{}` |
| `hostKeys` | Map of host keys (rsa, ecdsa, ed25519) | `{}` |
| `pdb.minAvailable` | PodDisruptionBudget configuration | `50%` |
| `initscripts` | Map of script names to content | `{}` |
| `ssh.sshd_config` | Custom sshd_config content | (see values.yaml) |
| `ssh.banner` | SSH login banner content | (see values.yaml) |
| `topologySpreadConstraints` | Topology spread constraints | (configured for HA) |

## Architecture

This chart deploys a Deployment using the standard `alpine` image.
Upon startup, the pod executes an entrypoint script that:
1.  Installs `openssh-server`, `bash`, `shadow`, and `openssh-server-pam` via `apk`.
2.  Generates host keys if they are not provided via Secret.
3.  Creates user accounts and configures `.ssh/authorized_keys` based on the `users` value.
4.  Runs any provided `initscripts`.
5.  Starts `sshd` on port `10022`.

This approach ensures the container always has the latest packages and removes the need for maintaining a custom Docker image.
