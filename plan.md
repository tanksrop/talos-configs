k8s cluster creation workflow?:
* download latest talos iso with qemu guest agent and upload to proxmox
* create proxmox vm with (minimum 2 cores, 2GB ram
* create variables for important info
  * export CONTROL_PLANE_IP=("<control-plane-ip-1>" "<control-plane-ip-2>" "<control-plane-ip-3>")
  * export WORKER_IP=("<worker-ip-1>" "<worker-ip-2>" "<worker-ip-3>"...)
* create a dns entry for each control plane ip to the same domain e.g. kube.homelab.internal (only if u have a dns server set up)
* export the previous chosen domain (if set, else pick one node ip) as a variable e.g. export YOUR_ENDPOINT=kube.homelab.internal
* talosctl gen secrets -o secrets.yaml
* set variable for cluster name export CLUSTER_NAME=<your_cluster_name>
* talosctl gen config --with-secrets secrets.yaml $CLUSTER_NAME https://$YOUR_ENDPOINT:6443
* create patch.yaml with vip, allow scheduling and disabling cni
* apply patch file talosctl machineconfig patch controlplane.yaml --patch @patch.yaml --output controlplane.yaml
* clone talosconfig
  * mkdir -p ~/.talos
  * cp ./talosconfig ~/.talos/config
  * export TALOSCONFIG=~/.talos/config
* configure endpoints in talosctl talosctl config endpoint <control_plane_IP_1> <control_plane_IP_2> <control_plane_IP_3>
* after finished booting, bootstrap ONE CONTROL PLANE talosctl bootstrap --nodes <control-plane-IP>
* merge kubeconfig with ONE control plane talosctl kubeconfig --nodes <control-plane-IP>
*   kubectl config get-clusters
*   kubectl config set-cluster <clustername>
* test with kubectl get nodes
* apply cilium cni with helm
* install gateway api with kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml


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


patch.yaml:
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
  allowSchedulingOnControlPlanes: true

machine:
  network:
    interfaces:
      - deviceSelector:
          physical: true
        dhcp: true
        vip:
          ip: 192.168.0.240

script to loop through control plane ips to apply

for ip in "${CONTROL_PLANE_IP[@]}"; do
  echo "=== Applying configuration to node $ip ==="
  talosctl apply-config --insecure \
    --nodes $ip \
    --file controlplane.yaml
  echo "Configuration applied to $ip"
  echo ""
done

helm command for cilium:
helm install \
    cilium \
    cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set=gatewayAPI.enabled=true \
    --set=gatewayAPI.enableAlpn=true \
    --set=gatewayAPI.enableAppProtocol=true


below is a working gatewayapi route to homer:

ip-pool.yaml
ben in ~/Documents/talos/test λ cat ippool.yaml 
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: vip-pool
spec:
  blocks:
  - cidr: "192.168.0.240/32"

gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: homer-gateway
  namespace: default
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same

httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homer-route
  namespace: default
spec:
  parentRefs:
  - name: homer-gateway
    namespace: default
  hostnames:
  - "test.internal"
  rules:
  - backendRefs:
    - name: homer
      port: 80
