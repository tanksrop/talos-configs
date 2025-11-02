k8s cluster creation workflow?:
* download latest talos iso with qemu guest agent and upload to proxmox
* create proxmox vm with (minimum 2 cores, 2GB ram)
* create variables for important info
  ```bash
    export CONTROL_PLANE_IP=("<control-plane-ip-1>" "<control-plane-ip-2>" ...)
    export WORKER_IP=("<worker-ip-1>" "<worker-ip-2>" ...) # if required
    export YOUR_ENDPOINT=kube.homelab.internal #can use an ip instead
  ```
* create a dns entry for each control plane ip to the same domain if using one
  
* Generate config files for talos and patch to use vip, allow scheduling on control planes and disable cni also clone talosconfig and set endpoints
  ```bash
  talosctl gen secrets -o secrets.yaml
  talosctl gen config --with-secrets secrets.yaml $CLUSTER_NAME https://$YOUR_ENDPOINT:6443
  talosctl machineconfig patch controlplane.yaml --patch @patch.yaml --output controlplane.yaml

  mkdir -p ~/.talos
  cp ./talosconfig ~/.talos/config
  export TALOSCONFIG=~/.talos/config

  talosctl talosctl config endpoint <control_plane_IP_1> <control_plane_IP_2> ...
  ```
* apply configs to nodes
  * control planes 
  ```bash
  for ip in "${CONTROL_PLANE_IP[@]}"; do
    echo "=== Applying configuration to node $ip ==="
    talosctl apply-config --insecure \
      --nodes $ip \
      --file controlplane.yaml
    echo "Configuration applied to $ip"
    echo ""
  done
  ```
* after finished booting, bootstrap ONE CONTROL PLANE
  ```bash
  talosctl bootstrap --nodes <control-plane-IP>
  ```
* merge kubeconfig with ONE control plane and test
  ```bash
  talosctl kubeconfig --nodes <control-plane-IP>

  kubectl config get-clusters
  kubectl config set-cluster <clustername>
  kubectl get nodes
  ```
* install gateway api
  ```bash
  kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
  ```
* apply cilium cni
  ```bash
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
      --set gatewayAPI.enabled=true \
      --set gatewayAPI.enableAlpn=true \
      --set gatewayAPI.enableAppProtocol=true \
      --set l2announcements.enabled=true
  ```
