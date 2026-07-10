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
