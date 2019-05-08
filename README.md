# k8s_master_migration
Kubernetes Bare Metal Master Node migration

Your old master node has crashed, is inaccessible or is simply getting old and you want to migrate to a new server with a new IP address.

This topic has been explored in this Github Issue: https://github.com/kubernetes/kubeadm/issues/338
As well as here: https://elastisys.com/2018/12/10/backup-kubernetes-how-and-why/


Here's my quick recap.

1. Backup the master (etcd and keys)

```
ssh $old_master_node
mkdir backup

# backup keys and certificates
sudo cp -r /etc/kubernetes/pki backup/

# backup etcd
sudo docker run --rm -v $(pwd)/backup:/backup \
    --network host \
    -v /etc/kubernetes/pki/etcd:/etc/kubernetes/pki/etcd \
    --env ETCDCTL_API=3 \
    k8s.gcr.io/etcd-amd64:3.2.18 \
    etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
    snapshot save /backup/etcd-snapshot-latest.db
```

2. Restore
```
ssh $new_master_node

scp -r $old_master_node:/home/$USER/backup .

# Restore certificates
sudo cp -r backup/pki /etc/kubernetes/

# Restore etcd backup
sudo mkdir -p /var/lib/etcd
sudo docker run --rm \
    -v $(pwd)/backup:/backup \
    -v /var/lib/etcd:/var/lib/etcd \
    --env ETCDCTL_API=3 \
    k8s.gcr.io/etcd-amd64:3.2.18 \
    /bin/sh -c "etcdctl snapshot restore '/backup/etcd-snapshot-latest.db' ; mv /default.etcd/member/ /var/lib/etcd/"

```

3. Find and update certificates referencing the old IP
```
cd /etc/kubernetes/pki
sudo su
for f in $(find -name "*.crt"); do sudo openssl x509 -in $f -text -noout > $f.txt; done
grep -R0 $old_ip  .

# delete matches
rm apiserver.crt apiserver.key
rm etcd/peer.crt etcd/peer.key
rm etc/server.crt etcd/server.key

# cleanup
for f in $(find -name "*.crt"); do rm $f.txt; done

# regenerate missing certs
sudo kubeadm init phase certs apiserver
sudo kubeadm init phase certs etcd-peer
sudo kubeadm init phase certs etcd-server

```

4. Start the master node
```
sudo kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

5. Join the cluster
```
# connect to all your cluster nodes
sudo kubeadm reset
sudo kubeadm join XXXXXXXXXX:6443 --token XXXXXX     --discovery-token-ca-cert-hash sha256:XXXXXX 

```


