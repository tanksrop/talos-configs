k8s cluster creation workflow?:
* install talosctl on pc
* talosctl gen config {cluster name} https://{ip for VIP}:6443
* clone controlplane.yaml to controlplane_worker.yaml and add taint to allow workloads and activate VIP and change cni to cilium
* talosctl apply-config --insecure --nodes {list of nodes} --file {role to apply}
* add secrets with credentials needed to access bitwarden secrets
* bootstrap argocd with helm and git repo and user already populated

talos-configs/  
├── machines/          # Talos machine configuration files (possibly not due to security risks)  
├── bootstrap/         # Scripts or configs for initial cluster bootstrap, automate future rebuilds  
├── manifests/         # Kubernetes manifests (argocd will point here)  
├── docs/              # Markdown docs (troubleshooting/maintainance and other info i might find useful in the future)  
├── README.md  
