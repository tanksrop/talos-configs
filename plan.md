k8s cluster creation workflow?:
* install talosctl on pc
* talosctl gen config {cluster name} https://{ip for VIP}:6443
* clone controlplane.yaml to controlplane_worker.yaml and add taint to allow workloads and activate VIP and change cni to cilium
* talosctl apply-config --insecure --nodes {list of nodes} --file {role to apply}
* add secrets with credentials needed to access bitwarden secrets
* bootstrap argocd with helm and git repo and user already populated

talos-configs/
├── machines/          # Talos machine configuration files (controlplane.yaml, worker.yaml, etc.)
├── bootstrap/         # Scripts or configs for initial cluster bootstrap
├── manifests/         # Kubernetes manifests (deployments, services, PVs, PVCs, ingress)
├── docs/              # Markdown docs (networking.md, ingress-migration.md, storage.md, etc.)
├── README.md
