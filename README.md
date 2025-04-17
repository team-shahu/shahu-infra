# shahu-infra
このレポジトリはしゃふKubernetesを支えるGitOpsリポジトリです。  

## 初期構築
### k3s
k3sをインストール
```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
  
Helmをインストール
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
  
#### Optional
必要があれば、外部からのアクセスを許可するために、自己証明書にIPを追加する
```bash
sudo kubectl -n kube-system edit secrets/k3s-serving
```
```
    listener.cattle.io/cn-10.43.0.1: 10.43.0.1
    listener.cattle.io/cn-127.0.0.1: 127.0.0.1
    listener.cattle.io/cn-100.99.125.62: 100.99.125.62  // この行追加
```
`100.99.125.62`はtailscaleのv4アドレスに置き換える。  
  
名前解決のため、DNSを別途指定する  
```bash
echo "nameserver 1.1.1.1" | sudo tee /etc/k3s-resolv.conf
echo 'kubelet-arg:' | sudo tee -a /etc/rancher/k3s/config.yaml
echo '- "resolv-conf=/etc/k3s-resolv.conf"' | sudo tee -a /etc/rancher/k3s/config.yaml
```
    
一旦k3sを止める  
```bash
sudo systemctl stop k3s
```
証明書作り直し  
```bash
sudo k3s certificate rotate
```
k3s 開始  
```bash
sudo systemctl start k3s
```
  
  
### Cloudflare オリジン証明書の追加
Cloudflareからオリジン間のセキュアな通信(full-strict)を実現するため、Cloudflareからオリジン証明書を取得する必要がある  
詳細は[こちらを参照](https://qiita.com/github0013@github/items/362d01b0ffb1eb4d3efb#%E5%A4%9A%E5%88%86%E4%B8%80%E7%95%AA%E7%B0%A1%E5%8D%98%E3%81%AA%E3%82%84%E3%82%8A%E6%96%B9)  
  
取得した証明書と秘密鍵をそれぞれのファイルに保存し、secretを作成
```
kubectl create secret tls cluster-tls --key priv.key --cert server.crt
```
  
  
### Longhorn
```
sudo apt -y install open-iscsi
```
LongHorn本体はArgoCD経由でデプロイされる
> [!NOTE]
> シングルノード構成の場合、 `Replica Node Level Soft Anti-Affinity`をwebコンパネ上から変更する必要がある。
> https://github.com/longhorn/longhorn/discussions/5737#discussioncomment-5587195
  
### SealedSecret
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/controller.yaml
```
${VERSION} は [SealedSecretのリポジトリ](https://github.com/bitnami-labs/sealed-secrets/releases/)の最新を使う。  
マスターキーのリストア  
```bash
# 鍵のSecretを生成
kubectl apply -f sealed-secrets-key.yaml

# Secretを生成しただけでは反映されないので、ControllerのPodを再生成する
# このPodはDeploymentによって生成されているので、Podを削除するとすぐに生成される
kubectl delete pods -n kube-system -l name=sealed-secrets-controller

# ControllerのPodが作り直されたことを確認
kubectl get pods -n kube-system -l name=sealed-secrets-controller
#NAME                                         READY   STATUS    #RESTARTS   AGE
#sealed-secrets-controller-867447b788-zhrg4   1/1     Running   #0          18s
```
  
> [!NOTE]
> バックアップ時は
> ```bash
> kubectl get secret -n kube-system \
> -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
> -o yaml > sealed-secrets-key.yaml
> ```
> で保存しておく。
  

#### ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm -rf argocd-linux-amd64
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Ingressの作成
```bash
nano argocd-ingress.yaml
```
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.shahu.ski`) && PathPrefix(`/`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: Host(`argocd.shahu.ski`) && PathPrefix(`/`) && Headers(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default
```
```bash
kubectl apply -f argocd-ingress.yaml
```
