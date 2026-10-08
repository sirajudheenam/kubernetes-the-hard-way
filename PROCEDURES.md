
```bash

ssh-keygen -t rsa

# copy the ssh keys to jumpbox
cat ~/.ssh/id_rsa.pub | ssh parallels@10.211.55.4 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# copy the ssh keys to k8s-server
cat ~/.ssh/id_rsa.pub | ssh parallels@10.211.55.7 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# copy the ssh keys to k8s-node-0
cat ~/.ssh/id_rsa.pub | ssh parallels@10.211.55.8 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# copy the ssh keys to k8s-node-1
cat ~/.ssh/id_rsa.pub | ssh parallels@10.211.55.9 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# make entries in the host file
jumpbox    10.211.55.4
k8s-server 10.211.55.7
k8s-node-0 10.211.55.8
ks8-node-1 10.211.55.9


# On all machines.
sudo su - 
passwd 
Enter new password and conform the same.

cat ~/.ssh/id_rsa.pub | ssh root@10.211.55.7 "sudo mkdir -p /root/.ssh && cat >> /root/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh root@10.211.55.8 "sudo -S mkdir -p /root/.ssh && cat >> /root/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh root@10.211.55.9 "sudo -S mkdir -p /root/.ssh && cat >> /root/.ssh/authorized_keys"

while read IP FQDN HOST SUBNET; do    ssh -n root@${IP} uname -o -m; done < machines.txt


while read IP FQDN HOST SUBNET; do 
    CMD="sed -i 's/^127.0.1.1.*/127.0.1.1\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n root@${IP} "$CMD"
    ssh -n root@${IP} hostnamectl set-hostname ${HOST}
done < machines.txt


for host in server node-0 node-1
   do ssh root@${host} uname -o -m -n
done

while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt



certs=(
  "admin" "node0" "node1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)

for i in ${certs[*]}; do
  openssl genrsa -out "${i}.key" 4096

  openssl req -new -key "${i}.key" -sha256 \
    -config "ca.conf" -section ${i} \
    -out "${i}.csr"
  
  openssl x509 -req -days 3653 -in "${i}.csr" \
    -copy_extensions copyall \
    -sha256 -CA "ca.crt" \
    -CAkey "ca.key" \
    -CAcreateserial \
    -out "${i}.crt"
done


for host in node0 node1; do
  ssh root@$host mkdir /var/lib/kubelet/
  
  scp ca.crt root@$host:/var/lib/kubelet/
    
  scp $host.crt \
    root@$host:/var/lib/kubelet/kubelet.crt
    
  scp $host.key \
    root@$host:/var/lib/kubelet/kubelet.key
done


scp \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:~/


for host in node0 node1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done


for host in node0 node1; do
  ssh root@$host "mkdir /var/lib/{kube-proxy,kubelet}"
  
  scp kube-proxy.kubeconfig \
    root@$host:/var/lib/kube-proxy/kubeconfig \
  
  scp ${host}.kubeconfig \
    root@$host:/var/lib/kubelet/kubeconfig
done


## configs/encryption-config.yaml is missing

cat > configs/encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF



# 09 

for host in node0 node1; do
  SUBNET=$(grep $host machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf 
    
  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml
    
  scp 10-bridge.conf kubelet-config.yaml \
  root@$host:~/
done


for host in node0 node1; do
  scp \
    downloads/runc.arm64 \
    downloads/crictl-v1.28.0-linux-arm.tar.gz \
    downloads/cni-plugins-linux-arm64-v1.3.0.tgz \
    downloads/containerd-1.7.8-linux-arm64.tar.gz \
    downloads/kubectl \
    downloads/kubelet \
    downloads/kube-proxy \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kubelet-config.yaml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@$host:~/
done

for host in node-0 node-1; do
  scp \
    downloads/runc.arm64 \
    downloads/crictl-v1.28.0-linux-arm.tar.gz \
    downloads/cni-plugins-linux-arm64-v1.3.0.tgz \
    downloads/containerd-1.7.8-linux-arm64.tar.gz \
    downloads/kubectl \
    downloads/kubelet \
    downloads/kube-proxy \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@$host:~/
done



```
