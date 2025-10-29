# talos-configs
This is going to be a repo to store the configuration for migrating my kubernetes cluster from k3s to talos
## cluster overview
* Servers: Currently 2 Virtualised nodes on a single proxmox host 
* Base OS: Talos Linux
* Kubernetes Distribution: Vanilla upstream k8s (managed by Talos)
* CNI: Cilium
* Ingress: Cilium native (replacing Traefik)
* Persistent Storage: NFS shares provided by TrueNAS
* GitOps: ArgoCD bootstrapped with Helm and preloaded Git repo
* Secrets Management: Bitwarden (fetched by workloads post-deploy)

## how i created the cluster and initial bootstrap
(Placeholder — document exact steps once finalized, possibly automated script in a bootstrap directory to speed up process of rebuilding entire cluster?)
## how do i add or remove nodes
(Placeholder — document exact steps once finalized)
## other docs i may need in the future
(Placeholder — common steps for troubleshooting cluster/node/pods?)
(Placeholder — document exact steps once finalized)
