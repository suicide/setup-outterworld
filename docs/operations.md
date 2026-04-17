# Operations

## Host model

Each VPS is its own single-node K3s cluster.

- SSH stays on port `22` by default unless overridden per host.
- K3s API port `6443` is not opened publicly.
- Public ingress is exposed only on `80` and `443`.
- Extra NodePorts are opened only from explicit per-host allowlists.

## kube-apiserver access via SSH tunnel

Create a local tunnel to the remote K3s API:

```bash
ssh -L 6443:127.0.0.1:6443 outterworld3
```

Then use a kubeconfig that points to `https://127.0.0.1:6443`.

## Flux

Each cluster runs Flux and reconciles its cluster entrypoint under `clusters/<hostname>/`, which in turn includes the shared `infrastructure/` manifests.

Ansible bootstraps a `flux-system/cluster-vars` Secret from `secrets/letsencrypt-email`.

## cert-manager

Flux installs cert-manager via a HelmRelease and creates two issuers:

- `letsencrypt-staging`
- `letsencrypt-prod`

Example ingress annotations:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - app.example.com
      secretName: app-example-com-tls
```

## Existing hosts

Existing manually provisioned VPSes can be added later by:

1. adding an inventory entry and host vars
2. skipping `playbooks/bootstrap.yml` because the admin user already exists
3. running `playbooks/site.yml`
