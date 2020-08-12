# multi-etcd-restore
Steps to restore etcd snapshot across a cluster of 3

## Considering your etcd servers are hosted on 3 master nodes with IPs(hostname): `10.204.14.87(controller-0)`,`10.204.14.180(controller-1)`,`10.204.14.165(controller-2)`

  ### Create a sample deployment
  
  `kubectl create deploy myetcd-test --image=busybox`

  ### On any controller
  
  `ETCDCTL_API=3 etcdctl snapshot save --cacert=/var/lib/kubernetes/ca.pem --cert=/var/lib/kubernetes/kubernetes.pem --key=/var/lib/kubernetes/kubernetes-key.pem newsnap.db`

  ### Delete the sample deploy, we'll try to restore this using etcd restore
  
  `kubectl delete deploy myetcd-test`
  
  ### Pull snapshot file from controller-0 on host/bastion machine
  
  `lxc file pull controller-0/root/newsnap.db .`
  
  ### Push snapshot file to controller-1
  
  `lxc file push newsnap.db controller-1/root/newsnap.db`
  
  ### Push snapshot file to controller-2
  
  `lxc file push newsnap.db controller-2/root/newsnap.db`
  
  
  ### On controller-0:
  
  `ETCDCTL_API=3 etcdctl snapshot restore --cacert=/var/lib/kubernetes/ca.pem --cert=/var/lib/kubernetes/kubernetes.pem --key=/var/lib/kubernetes/kubernetes-key.pem --data-dir=/var/lib/etcd-restored-3 --name=controller-0   --initial-cluster controller-0=https://10.204.14.87:2380,controller-1=https://10.204.14.180:2380,controller-2=https://10.204.14.165:2380 --initial-cluster-token=etcd-restored-3 --initial-advertise-peer-urls=https://10.204.14.87:2380 newsnap.db`
  
  
  ### On controller-1:
  
  `ETCDCTL_API=3 etcdctl snapshot restore --cacert=/var/lib/kubernetes/ca.pem --cert=/var/lib/kubernetes/kubernetes.pem --key=/var/lib/kubernetes/kubernetes-key.pem --data-dir=/var/lib/etcd-restored-3 --name=controller-1   --initial-cluster controller-0=https://10.204.14.87:2380,controller-1=https://10.204.14.180:2380,controller-2=https://10.204.14.165:2380 --initial-cluster-token=etcd-restored-3 --initial-advertise-peer-urls=https://10.204.14.180:2380 newsnap.db`
  
  
  ### On controller-2:
  
  `ETCDCTL_API=3 etcdctl snapshot restore --cacert=/var/lib/kubernetes/ca.pem --cert=/var/lib/kubernetes/kubernetes.pem --key=/var/lib/kubernetes/kubernetes-key.pem --data-dir=/var/lib/etcd-restored-3 --name=controller-2   --initial-cluster controller-0=https://10.204.14.87:2380,controller-1=https://10.204.14.180:2380,controller-2=https://10.204.14.165:2380 --initial-cluster-token=etcd-restored-3 --initial-advertise-peer-urls=https://10.204.14.165:2380 newsnap.db`
  
  ### On all 3 controllers, edit etcd.service to point to new token(etcd-restoed-3, in this case), also data directory to /var/lib/etcd-restored-3 in this case
  
  `vi /etc/systemd/system/etcd.service`
  `systemctl daemon-reload`
  `systemctl restart etcd`
  
  ### Verify from any controller
  
  `ETCDCTL_API=3 etcdctl member list --cacert=/var/lib/kubernetes/ca.pem --cert=/var/lib/kubernetes/kubernetes.pem --key=/var/lib/kubernetes/kubernetes-key.pem`

  ### Verify whether the deleted deployment respawned or not
  
  `kubectl get deploy`
  
