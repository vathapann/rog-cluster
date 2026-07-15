# rog-cluster
My old laptop cluster for my Kubernetes Homelab Exploration

# Commands learn along the way

Use Markdown code blocks for commands. For shell commands, wrap them with triple backticks and specify `bash` or `sh`:

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

*How to find group id?
```bash
    kubectl logs -n linkding-{hash-number} | grep set 
```

## Problem: cloudflared pods stuck as `Unknown` in k9s (SOPS + Flux)

Symptom: `cloudflared` pods in the `linkding` namespace showed `Unknown` status in k9s
and kept restarting.

### Cause 1 — encrypted secret manifest wasn't wired into any kustomization

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

### Cause 2 — whole-document SOPS encryption isn't compatible with Flux

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
`data`/`stringData` values — so it rejected the file.

Fix: re-encrypt with an explicit `--encrypted-regex` so only the secret payload is
ciphertext:

```bash
sops -d tunnel-credentials.yaml > plain.yaml
sops --encrypt --age <age-public-key> \
  --encrypted-regex '^(data|stringData)$' \
  --input-type yaml --output-type yaml \
  plain.yaml > tunnel-credentials.yaml
```

### Lessons learned

- Flux reporting a Kustomization as "Applied" for a commit doesn't mean every file in
  that commit was actually used — check it's genuinely listed in a `kustomization.yaml`
  resources tree that Flux syncs.
- `.sops.yaml`'s `path_regex`/`encrypted_regex` rules are resolved relative to the
  target file's location at encryption time. Encrypting a file outside any
  `.sops.yaml`'s reach silently falls back to full-document encryption.
- `sops -d` succeeding locally does **not** guarantee Flux can decrypt the same file —
  Flux has stricter expectations (plaintext `apiVersion`/`kind`/`metadata`) than the
  raw `sops` CLI.
- Useful debugging commands:
  ```bash
  kubectl describe pod -n <ns> <pod>                       # mount/volume errors
  kubectl get kustomization -n flux-system apps -o jsonpath='{.status.conditions}'
  kubectl logs -n flux-system deploy/kustomize-controller --tail=80
  ```
