# k8s-ssh-bastion

A lightweight, secure SSH bastion for Kubernetes, designed to provide SSH access to your cluster network.

## Features

-   **Standard Base Image**: Uses the official `ubuntu:24.04` image.
-   **Runtime Installation**: Installs `openssh-server` at pod startup, ensuring latest security updates.
-   **Persistent Host Keys**: Supports providing your own host keys or automatically generating them via Helm (persisted across upgrades).
-   **User Management**: easily manage users and their public keys via Helm values.
-   **Secure Defaults**: configured with security best practices (no password auth, no root login by default).

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

Add users and their public keys in `values.yaml`. The keys will be mounted to `/etc/ssh/authorized_keys/<username>`.

```yaml
users:
  alice: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
  bob: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA..."
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
| `image.repository` | Image repository | `ubuntu` |
| `image.tag` | Image tag | `24.04` |
| `replicaCount` | Number of replicas | `1` |
| `service.type` | Service type | `NodePort` |
| `service.nodePort` | Node port (if type is NodePort) | `30022` |
| `users` | Map of usernames to public keys | `{}` |
| `hostKeys` | Map of host keys (rsa, ecdsa, ed25519) | `{}` |
| `ssh.sshd_config` | Custom sshd_config content | (see values.yaml) |

## Architecture

This chart deploys a Deployment using the standard `ubuntu:24.04` image.
Upon startup, the pod executes an entrypoint script that:
1.  Installs `openssh-server` and `zsh` via `apt-get`.
2.  Generates ephemeral host keys if they are not provided via Secret.
3.  Creates user accounts based on the provided `users` configuration.
4.  Starts `sshd`.

This approach ensures the container always has the latest packages and removes the need for maintaining a custom Docker image.
