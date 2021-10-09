## ip-change-for-edge-kubernetes

### 【前提】   
Kubernetesでは、各Node のIP が変わるとクラスター全体、または、それぞれの Node が機能しなくなります。  
従い、Kubernetesは、固定IPが必須となります。一方で、エッジ環境においては、よりIP変更の影響を受けやすいです。   
ここでは、特にエッジ環境においてIPの変更が必要になった場合の手順を説明します。

### 【Kubernetes IP変更手順 概要】   
旧IPを参照している証明書及びシステムコンポーネントの設定を変更し、kubeletとdockerを再起動します。    

### 【Kubernetes IP変更手順 詳細】  
1.旧IPを参照しているシステムコンポーネントを更新します。 
  
```
sudo su -
oldip=<旧IP>
newip=<新IP>
cd /etc/kubernetes
find . -type f | xargs grep $oldip
find . -type f | xargs sed -i "s/$oldip/$newip/"
find . -type f | xargs grep $newip
```

2.旧IPを参照している証明書を入れ替えます。   
  
```
cd /etc/kubernetes/pki
rm apiserver.crt apiserver.key
kubeadm init phase certs apiserver --apiserver-advertise-address=$newip
rm etcd/peer.crt etcd/peer.key
kubeadm init phase certs etcd-peer
```

3.kubelet、dockerを再起動します。   
   
```
systemctl restart kubelet
systemctl restart docker
exit
```
   
4.以下コマンドを実行し、旧IPになっている箇所を新IPに置き換えます。   
   
```
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl -n kube-system edit cm kubeadm-config
kubectl -n kube-system edit cm kube-proxy
```
  
5.calicoをインストールします。（入っていない場合のみ）   
   
```
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/xxxxxxxxxx/Documentation/kube-flannel.yml
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

6.ネットワークプラグインとしてflannelがインストールされている場合、calicoに置き換えます。   
  
```   
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/xxxxxxxxxxxx/Documentation/kube-flannel.yml
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```   
flannelの場合、IPを更新するとエラーで落ちてしまいます。   
k8s communityにおいてもflannelは非推奨なので、ネットワークプラグインとしてcalicoを採用しています。  