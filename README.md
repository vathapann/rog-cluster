# rog-cluster

My old laptop cluster, repurposed as a Kubernetes homelab.

I'd mostly used Docker and Docker Compose for deployments, but wanted to explore what Kubernetes offers in terms of self-healing and fault tolerance. K3s was the natural entry point for getting started.

This repo documents what I'm learning along the way such as Kubernetes, GitOps with Flux, secrets management with SOPS, exposing services with Cloudflare Tunnel, and monitoring with Prometheus/Grafana.

## Architecture & Concepts

### Why GitOps with Flux (vs. traditional CI/CD)

I have some experiences with CI/CD and understand that it is a push-based approach which mean the pipline builds an artifact (built source codes) and push it into the cluster. So the CI would needs credentials to do write access to our infrastructure. We can confirm this by setting a value and varialbe in github secret section. 

At the same time, I saw GitOps was really popular. So I gave it a try by using GitOps with Flux. Basically, GitOps works inverstly to CI/CD. It is a pull-based approach which mean it sits in the cluster and listen for commit changes at some interval. So our credentials are never go outside of the cluster.

How Flux works and monitors changes:
- `clusters/staging/apps.yaml` and `clusters/staging/monitoring.yaml` are Flux
  `Kustomization` objects that tell `kustomize-controller` which paths to sync
  (`./apps/staging`, `./monitoring/controllers/staging`) and how often (every `1m`/`30m`).
 See the
  `driftDetection` block in [release.yaml](monitoring/controllers/base/kube-prometheus-stack/release.yaml)
  for an example of that guarantee in action.
- Git becomes the single source of truth and the audit log. `git log` shows exactly
  what changed and when, instead of digging through CI run history.

### Exposing the homelab with Cloudflare Tunnel

The cluster sits behind my home network with no public IP and no port-forwarding. Instead
of exposing anything directly, [cloudflare.yaml](apps/staging/linkding/cloudflare.yaml)
runs a `cloudflared` deployment that opens an *outbound* connection from inside the
cluster to Cloudflare. So there's no inbound port to secure.

- The tunnel (`ldrog`) is configured entirely via a `ConfigMap`, mapping a public
  hostname to an internal service: `homelab.sovathapann.site` → `http://linkding:9090`.
- Authentication is a credentials file, stored as a SOPS-encrypted `Secret`
  (`tunnel-credentials.yaml`) and decrypted by Flux at apply time.
- `replicas: 2` gives basic redundancy for the tunnel itself.
- See the [Troubleshooting Log](#learnings--troubleshooting-log) below for the SOPS +
  Flux that I struggling getting it running.


### Linkding as a POC app

[Linkding](https://github.com/sissbruecker/linkding) (a self-hosted bookmark manager) is
the first real app deployed on this cluster. It is chosen because it's simple enough to just prove end-to-end without putting all the sweat into development work.

- A `Deployment` running a single container ([deployment.yaml](apps/base/linkding/deployment.yaml)),
  with a hardened `securityContext` (`runAsUser`/`runAsGroup: 33`, `allowPrivilegeEscalation: false`).
- Persistent state via a `PersistentVolumeClaim` ([storage.yaml](apps/base/linkding/storage.yaml)),
  so I could learn how storage survives pod restarts/rescheduling.
- Config and credentials injected through a `Secret` (`linkding-container-env-secret.yaml`),
  encrypted with SOPS.
- Exposed internally via a `Service`, then externally through both `Ingress` (Traefik)
  and Cloudflare Tunnel.

Having one small app exercise the whole path. From  Deployment → Storage → Secret → Service →
Ingress/Tunnel, all reconciled by Flux. This made it a good proving ground for me to start with.

### Monitoring with kube-prometheus-stack (Helm)

Prometheus + Grafana are installed as a single Flux-managed `HelmRelease`
([release.yaml](monitoring/controllers/base/kube-prometheus-stack/release.yaml)), synced
via the `monitoring` Kustomization (`./monitoring/controllers/staging`, reconciled every
`1m`). Flux pulls the chart from a `HelmRepository` source and manages install/upgrade 
including CRDs (`install.crds: Create`, `upgrade.crds: CreateReplace`).

Grafana is exposed at `grafana.sovathapann.site` through the same Traefik ingress class
used elsewhere in the cluster.

### Ingress with Traefik

Ingress in this cluster uses `ingressClassName: traefik`, the ingress controller that
ships built into K3s by default. `linkding`'s
[ingress.yaml](apps/staging/linkding/ingress.yaml) is the simplest example: it routes
`homelab.sovathapann.site` to the `linkding` service on port `9090`.

<!-- TODO: if/when you install a standalone Traefik (or swap to ingress-nginx) via its
     own Helm chart instead of relying on K3s's bundled one, document the "why" and the
     HelmRelease here, same pattern as kube-prometheus-stack above. -->

## Learnings & Troubleshooting Log

How to check for change in the deployment using flux:
```bash
flux get kustomizations 
```


Refer to Security Measure in containers of linkding:

We can set privilege permission on linkding container by specify userid in the manifest file.

```yaml
    securityContext:
        runAsUser: 33 # www-data group ID
        runAsGroup: 33
        fsGroup: 33
```

How to find group id?
```bash
    kubectl logs -n linkding-{hash-number} | grep set 
```
### How flux find work under the hood in this project
1. flux-system Kustomization (path: clusters/staging)
2. applies clusters/staging/apps.yaml
3. creates "apps" Kustomization (path: apps/staging)
4. kustomize build renders apps/staging/linkding/
5. pulls in apps/base/linkding via `resources:`

### CRDs (Custom Resource Definition)

This is how kubernetes lets us extend its APIs whit our own kinds. (beyond Pod and Deployment).
E.g. in repository.yaml of 'monitoring/controllers/base' Flux creates 'helmrepositories.source.toolkit.fluxcd.io'

The repository.yaml file itself is one I wrote by hand. it's
an instance of that kind (a "custom resource"). Flux's source-controller is the
piece that actually watches for HelmRepository objects and does something with them. 

## Problem: cloudflared pods stuck as `Unknown` in k9s (SOPS + Flux)

Symptom: `cloudflared` pods in the `linkding` namespace showed `Unknown` status in k9s
and kept restarting.

#### Cause 1: encrypted secret manifest wasn't wired into any kustomization

`tunnel-credentials.yaml` (the SOPS-encrypted `Secret` for the tunnel) was committed at
the repo root, but Flux's `apps` Kustomization only syncs `./apps/staging`, and
`apps/staging/linkding/kustomization.yaml` never listed the file as a resource. Flux
happily reported the commit as applied, but the Secret was never actually created —
so `kubelet` couldn't mount it, killed/restarted the container in a loop, and k9s
showed `Unknown`.

Fix: move the file under `apps/staging/linkding/` and add it to that kustomization's
`resources:` list.

```bash
kubectl describe pod -n linkding <cloudflared-pod>
# Warning  FailedMount  ...  MountVolume.SetUp failed for volume "creds" : secret "tunnel-credentials" not found
```

#### Cause 2: whole-document SOPS encryption isn't compatible with Flux

Once wired in, Flux tried to decrypt the file for the first time and failed:

```text
decryption failed for ...: error decrypting sops tree: Error walking tree:
Could not decrypt value: Input string <redacted> does not match sops' data format
```

The file had been encrypted as a **whole document** (`apiVersion`, `kind`,
`metadata.name` all wrapped in `ENC[...]`), because it was originally encrypted
outside the scope of `clusters/staging/.sops.yaml` (whose `encrypted_regex` only
applies based on the file's path when you run `sops -e`). The standalone `sops` CLI
decrypts whole-document encryption fine, but **Flux's `kustomize-controller` requires
`apiVersion` / `kind` / `metadata` to stay plaintext** and only decrypts
`data`/`stringData` values, so it rejected the file.

Fix: re-encrypt with an explicit `--encrypted-regex` so only the secret payload is
ciphertext:

```bash
sops -d tunnel-credentials.yaml > plain.yaml
sops --encrypt --age <age-public-key> \
  --encrypted-regex '^(data|stringData)$' \
  --input-type yaml --output-type yaml \
  plain.yaml > tunnel-credentials.yaml
```

#### Lessons learned

- Flux reporting a Kustomization as "Applied" for a commit doesn't mean every file in
  that commit was actually used. It's genuinely listed in a `kustomization.yaml`
  resources tree that Flux syncs.
- `.sops.yaml`'s `path_regex`/`encrypted_regex` rules are resolved relative to the
  target file's location at encryption time. Encrypting a file outside any
  `.sops.yaml`'s reach silently falls back to full-document encryption.
- `sops -d` succeeding locally does **not** guarantee Flux can decrypt the same file.
  Flux has stricter expectations (plaintext `apiVersion`/`kind`/`metadata`) than the
  raw `sops` CLI.
- Useful debugging commands:
  ```bash
  kubectl describe pod -n <ns> <pod>                       # mount/volume errors
  kubectl get kustomization -n flux-system apps -o jsonpath='{.status.conditions}'
  kubectl logs -n flux-system deploy/kustomize-controller --tail=80
  ```
