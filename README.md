# neuvector_rke2
```bash
sudo bash -c 'curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL="v1.22" sh -'
```
sudo mkdir -p /etc/rancher/rke2
sudo bash -c 'echo "write-kubeconfig-mode: \"644\" " > /etc/rancher/rke2/config.yaml'
sudo systemctl enable --now rke2-server.service
sudo journalctl -u rke2-server
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
cat /etc/rancher/rke2/rke2.yaml
mkdir -p ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
kubectl get nodes
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version --client --short
helm ls --all-namespaces
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
--create-namespace \
--namespace cert-manager \
--version v1.7.1 \
--set installCRDs=true \
--set nodeSelector."kubernetes\.io/os"=linux \
--set webhook.nodeSelector."kubernetes\.io/os"=linux \
--set cainjector.nodeSelector."kubernetes\.io/os"=linux \
--set startupapicheck.nodeSelector."kubernetes\.io/os"=linux 
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
helm repo add neuvector https://neuvector.github.io/neuvector-helm/
helm repo update

cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF

cat <<EOF > ~/neuvector-values.yaml
k3s:
  enables: true
controller:
  replicas: 1
cve:
  scanner:
    replicas: 1
manager:
  ingress:
    enabled: true
    host: neuvector.172.17.203.173.sslip.io
    annotations:
      cert-manager.io/cluster-issuer: selfsigned-cluster-issuer
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    tls: true
    secretName: neuvector-tls-secret
EOF   

helm install neuvector neuvector/core \
--namespace cattle-neuvector-system \
-f ~/neuvector-values.yaml \
--version 2.2.0 \
--create-namespace

