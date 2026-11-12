# Operations

## Host model

Each VPS is its own single-node K3s cluster.

- SSH stays on port `22` by default unless overridden per host.
- K3s API port `6443` is not opened publicly.
- Public ingress is exposed only on `80` and `443`.
- Extra NodePorts are opened only from explicit per-host allowlists.

## AppArmor stacked-peer workaround

Ubuntu 26.04 hosts on kernel `7.0.0-42` or later stack the containerd profile
with `unconfined`. Container processes then carry
`cri-containerd.apparmor.d//&unconfined`, and containerd's default profile
cannot signal them. PostgreSQL cannot wake its checkpointer or backends.

The fix is a full replacement profile in
`/etc/apparmor.d/cri-containerd.apparmor.d`, plus the two stacked-peer rules.
Enable it per host with `cri_apparmor_stacked_peer_workaround: true` in that
host's `inventory/host_vars` file. It is currently enabled only on
`outterworld3`. The Debian nodes do not use AppArmor.

Apply it without a dist-upgrade or a K3s upgrade:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
  --limit outterworld3 --tags cri-apparmor
```

The role keeps both AppArmor restriction sysctls at `1`, loads the profile
before K3s, and confirms the profile is enforcing. Do not mark a pod AppArmor
`Unconfined`, do not disable the sysctl, and do not delete the PostgreSQL PVC.

The stuck postmaster does not recover in place. After the role succeeds, an
operator may recycle only the pod:

```bash
sudo k3s kubectl -n broadcaster-monitor delete pod broadcaster-monitor-postgresql-0
```

When a k3s release includes containerd PR 14487, set the flag to `false` and run
the same tagged command. The role removes the file. A profile that is already
loaded stays until the next boot, when `apparmor.service` no longer loads the
override and K3s starts with the runtime profile.

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

## Cutover

For the first migration on an existing cluster:

1. run `ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit <host>`
2. wait for Flux to report `GitRepository` and `Kustomization` as Ready
3. verify `kubectl get helmreleases -A` shows the `cert-manager` HelmRelease as Ready
4. verify `kubectl get clusterissuers` shows both issuers as Ready
5. verify `kubectl get pods -n cert-manager` stays healthy during the handoff

Recommended checks:

```bash
kubectl -n flux-system get gitrepositories,kustomizations
kubectl get helmreleases -A
kubectl get pods -n cert-manager
kubectl get clusterissuers
```

## Rollback

If Flux bootstrap succeeds but `cert-manager` does not reconcile cleanly:

1. delete the Flux `Kustomization` for the cluster with `kubectl -n flux-system delete kustomization cluster-config`
2. delete the Flux `GitRepository` with `kubectl -n flux-system delete gitrepository platform-config`
3. restore the previous Ansible-managed `cert-manager` and `cluster_issuers` flow from Git
4. rerun `playbooks/site.yml`

Do not delete the existing `cert-manager` namespace, ACME account secrets, or issued certificate secrets during rollback.

## Existing hosts

Existing manually provisioned VPSes can be added later by:

1. adding an inventory entry and host vars
2. skipping `playbooks/bootstrap.yml` because the admin user already exists
3. running `playbooks/site.yml`
