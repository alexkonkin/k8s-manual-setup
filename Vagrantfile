VAGRANTFILE_API_VERSION = "2"

$install_misc = <<-SCRIPT
  node_name=$1
  node_type=$2
  echo    ""
  echo    "---------------------------------------------------------------------"
  echo    "| Step : Install a set of handy packages ($node_name, $node_type) |"
  echo    "---------------------------------------------------------------------"
  echo    ""

  yum install -y mc crudini glances sshpass
SCRIPT

$configure_hosts = <<-SCRIPT
  node_name=$1
  node_type=$2

  echo    ""
  echo    "--------------------------------------------------------"
  echo    "| Step : Configure hosts  ( $node_name, $node_type)    |"
  echo    "---------------------------------------------------------"
  echo    ""

cat <<EOF | sudo tee /etc/hosts.new
172.16.94.10                    c1-master1
172.16.94.11                    kplabs-cka-worker
172.16.94.12                    c1-etcd2
EOF
   
   sudo rm -fv /etc/hosts
   sudo mv -v /etc/hosts.new /etc/hosts
SCRIPT

$disable_swap = <<-SCRIPT
  node_name=$1
  node_type=$2

  echo    ""
  echo    "--------------------------------------------------------"
  echo    "| Step : Disable Swap  ( $node_name, $node_type)    |"
  echo    "---------------------------------------------------------"
  echo    ""

  sed -e '/.*swap.*/ s/^#*/#/' -i /etc/fstab
  swapoff -a

SCRIPT

$download_binaries_srv = <<-SCRIPT
  node_name=$1
  node_type=$2

  echo    ""
  echo    "----------------------------------------------------------"
  echo    "| Step : Download Server Binaries ($node_name, $node_type)|"
  echo    "----------------------------------------------------------"
  echo    ""

  yum -y install  wget
  mkdir /root/binaries
  cd /root/binaries
  wget https://dl.k8s.io/v1.18.0/kubernetes-server-linux-amd64.tar.gz
  tar -xzvf kubernetes-server-linux-amd64.tar.gz
  cd /root/binaries/kubernetes/server/bin/
  wget https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz
  tar -xzvf etcd-v3.4.10-linux-amd64.tar.gz
SCRIPT

$download_binaries_node = <<-SCRIPT
  node_name=$1
  node_type=$2

  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Download Node Binaries ($node_name, $node_type)       |"
  echo    "-----------------------------------------------------------------"
  echo    ""

  yum -y install  wget
  mkdir /root/binaries
  cd /root/binaries
  wget https://dl.k8s.io/v1.18.0/kubernetes-node-linux-amd64.tar.gz
  tar -xzvf kubernetes-node-linux-amd64.tar.gz
SCRIPT

$create_ca_cert = <<-SCRIPT
  node_name=$1
  node_type=$2

  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Create self signed sertificate ($node_name, $node_type) |"
  echo    "-----------------------------------------------------------------"
  echo    ""
  
  rm -rfv /root/certificates
  echo "---------------------------------------------------"
  echo "0. Create base directory were all the certificates and keys will be stored"
  echo "---------------------------------------------------"
  mkdir -pv /root/certificates
  cd /root/certificates

  echo "---------------------------------------------------"
  echo "1. Creating a private key for Certificate Authority"
  echo "---------------------------------------------------"
  
  openssl genrsa -out ca.key 2048
  
  echo "---------------------------------------------------"
  echo "2. Creating CSR certificate request"
  echo "---------------------------------------------------"
  
  openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

  echo "---------------------------------------------------"
  echo "3. Self-Sign the CSR for the local CA authority"
  echo "---------------------------------------------------"

  openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000

SCRIPT

$configure_etcd = <<-SCRIPT
  node_name=$1
  node_type=$2
  etcd_ip=$3

  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Configure and start ETCD ($node_name, $node_type) |"
  echo    "-----------------------------------------------------------------"
  echo    ""
 
  echo "---------------------------------------------------"
  echo "1. Set Selinux to permissive mode"
  echo "---------------------------------------------------"

  sudo su -
  setenforce 0
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

  echo "---------------------------------------------------"
  echo "2. Generate etcd key"
  echo "---------------------------------------------------"

  cd /root/certificates/
  openssl genrsa -out etcd.key 2048

  echo "---------------------------------------------------"
  echo "3. Create configuration file for etcd's certificate"
  echo "---------------------------------------------------"

cat > etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = $etcd_ip
IP.2 = 127.0.0.1
EOF

  echo "---------------------------------------------------"
  echo "4. Creating CSR"
  echo "---------------------------------------------------"
  
  openssl req -new -key etcd.key -subj "/CN=etcd" -out etcd.csr -config etcd.cnf

  echo "---------------------------------------------------"
  echo "5. Self-Sign the CSR"
  echo "---------------------------------------------------"

  openssl x509 -req -in etcd.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out etcd.crt -extensions v3_req -extfile etcd.cnf -days 1000

  echo "---------------------------------------------------"
  echo "6. Copy the Certificates and Key to /etc/etcd"
  echo "---------------------------------------------------"

  mkdir -v /etc/etcd
  cp -v etcd.crt etcd.key ca.crt /etc/etcd

  echo "---------------------------------------------------"
  echo "7. Copy the ETCD and ETCDCTL Binaries to the Path"
  echo "---------------------------------------------------"
  
  cd /root/binaries/kubernetes/server/bin/etcd-v3.4.10-linux-amd64/
  cp -v etcd etcdctl /usr/local/bin/

  echo "---------------------------------------------------"
  echo "8. Configure the Systemd File"
  echo "---------------------------------------------------"

  SERVER_IP=$etcd_ip

cat > /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name master-1 \\
  --cert-file=/etc/etcd/etcd.crt \\
  --key-file=/etc/etcd/etcd.key \\
  --peer-cert-file=/etc/etcd/etcd.crt \\
  --peer-key-file=/etc/etcd/etcd.key \\
  --trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-trusted-ca-file=/etc/etcd/ca.crt \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${SERVER_IP}:2380 \\
  --listen-peer-urls https://${SERVER_IP}:2380 \\
  --listen-client-urls https://${SERVER_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${SERVER_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster master-1=https://${SERVER_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

echo "---------------------------------------------------"
echo "9. Enable and start ETCD service"
echo "---------------------------------------------------"
systemctl start etcd
systemctl status etcd
systemctl enable etcd

echo "---------------------------------------------------"
echo "10. Testing ETCD service via https"
echo "---------------------------------------------------"
ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key put course "kplabs cka course is awesome"
ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key get course

SCRIPT

$backup_etcd = <<-SCRIPT
  node_name=$1
  node_type=$2
  etcd_ip=$3

  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Create backup of ETCD ($node_name, $node_type) |"
  echo    "-----------------------------------------------------------------"
  echo    ""
  
  export PATH=$PATH:/usr/local/bin
  rm -fv /vagrant/*.db

  ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key put backup_date $(date +%H:%M:%S_%d:%m:%Y)

  ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key snapshot save /tmp/snapshot.db

  etcdctl --write-out=table snapshot status /tmp/snapshot.db
  chown -v vagrant:vagrant /tmp/snapshot.db

SCRIPT

$restore_etcd = <<-SCRIPT
  node_name=$1
  node_type=$2
  etcd_ip=$3
  SERVER_IP=$4

  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Restore from backup of ETCD ($node_name, $node_type) |"
  echo    "-----------------------------------------------------------------"
  echo    ""
 
  su - vagrant -c 'cat /dev/zero | ssh-keygen -q -N ""'
  su - vagrant -c 'sshpass -p "'${password}'" ssh-copy-id -o StrictHostKeyChecking=no "'${SERVER_IP}'"'
  su - vagrant -c 'scp -o StrictHostKeyChecking=no -r "'${SERVER_IP}'":/tmp/snapshot.db /tmp/'

  export PATH=$PATH:/usr/local/bin
  cd /tmp
  rm -rfv mv /tmp/default.etcd

  ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key snapshot restore /tmp/snapshot.db
  
  systemctl stop etcd

  rm -rfv /var/lib/etcd/*

  mv /tmp/default.etcd/* /var/lib/etcd

  systemctl start etcd

  systemctl status etcd
  ETCDCTL_API=3 /usr/local/bin/etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd.crt --key=/etc/etcd/etcd.key get --from-key 0
  
SCRIPT

$configure_api_server = <<-SCRIPT
  node_name=$1
  node_type=$2
  SERVER_IP=$3
  
  sleep 1
  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "| Step : Configure and start API-SERVER ($node_name, $node_type) |"
  echo    "-----------------------------------------------------------------"
  echo    ""
  
  sleep 1
  echo "------------------------------------------------------------------------"
  echo "Pre:Requisite Step: Move the kube-apiserver binary to /usr/bin directory."
  echo "------------------------------------------------------------------------"

  cd /root/binaries/kubernetes/server/bin/
  cp -v kube-apiserver /usr/local/bin/
   
  sleep 1 
  echo "------------------------------------------------------------"
  echo "Step 1. Generate Configuration File for CSR Creation."
  echo "-------------------------------------------------------------"
  
  cd /root/certificates

cat <<EOF | sudo tee api.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = ${SERVER_IP}
IP.3 = 10.32.0.1
EOF

sleep 1 
echo "------------------------------------------------------------"
echo "Step 2: Generate Certificates for API Server"
echo "-------------------------------------------------------------"

openssl genrsa -out kube-api.key 2048
openssl req -new -key kube-api.key -subj "/CN=kube-apiserver" -out kube-api.csr -config api.conf
openssl x509 -req -in kube-api.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-api.crt -extensions v3_req -extfile api.conf -days 1000

sleep 1 
echo "------------------------------------------------------------"
echo "Step 3: Generate Certificate for Service Account:"
echo "-------------------------------------------------------------"

openssl genrsa -out service-account.key 2048
openssl req -new -key service-account.key -subj "/CN=service-accounts" -out service-account.csr
openssl x509 -req -in service-account.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out service-account.crt -days 100

sleep 1 
echo "-------------------------------------------------------------------"
echo "Step 4: Copy the certificate files to /var/lib/kubernetes directory"
echo "-------------------------------------------------------------------"

mkdir /var/lib/kubernetes
cp etcd.crt etcd.key ca.crt kube-api.key kube-api.crt service-account.crt service-account.key /var/lib/kubernetes

sleep 1 
echo "-------------------------------------------------------------------"
echo "Step 5: Creating Encryption key and Configuration"
echo "-------------------------------------------------------------------"

ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-at-rest.yaml <<EOF
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

cp -v encryption-at-rest.yaml /var/lib/kubernetes/encryption-at-rest.yaml

sleep 1 
echo "-------------------------------------------------------------------"
echo "Step 6: Creating Systemd service file:"
echo "-------------------------------------------------------------------"

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
--advertise-address=${SERVER_IP} \
--allow-privileged=true \
--authorization-mode=Node,RBAC \
--client-ca-file=/var/lib/kubernetes/ca.crt \
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
--enable-bootstrap-token-auth=true \
--etcd-cafile=/var/lib/kubernetes/ca.crt \
--etcd-certfile=/var/lib/kubernetes/etcd.crt \
--etcd-keyfile=/var/lib/kubernetes/etcd.key \
--etcd-servers=https://127.0.0.1:2379 \
--kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \
--kubelet-client-certificate=/var/lib/kubernetes/kube-api.crt \
--kubelet-client-key=/var/lib/kubernetes/kube-api.key \
--kubelet-https=true \
--service-account-key-file=/var/lib/kubernetes/service-account.crt \
--service-cluster-ip-range=10.32.0.0/24 \
--tls-cert-file=/var/lib/kubernetes/kube-api.crt \
--tls-private-key-file=/var/lib/kubernetes/kube-api.key \
--requestheader-client-ca-file=/var/lib/kubernetes/ca.crt \
--service-node-port-range=30000-32767 \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/log/kube-api-audit.log \
--bind-address=0.0.0.0 \
--event-ttl=1h \
--encryption-provider-config=/var/lib/kubernetes/encryption-at-rest.yaml \
--v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sleep 1 
echo "-------------------------------------------------------------------"
echo "Step 7: Start the kube-api service:"
echo "-------------------------------------------------------------------"
systemctl start kube-apiserver
systemctl status kube-apiserver
systemctl enable kube-apiserver

SCRIPT

$configure_controller_manager = <<-SCRIPT
  node_name=$1
  node_type=$2
  SERVER_IP=$3
  delay=$4 

  echo "------------------------------------------------------------"
  echo "Step 1: Generate Certificates for Controller Manager"
  echo "-------------------------------------------------------------"
  sleep $delay

  export PATH=$PATH:/usr/local/bin

  cd /root/certificates

  openssl genrsa -out kube-controller-manager.key 2048
  openssl req -new -key kube-controller-manager.key -subj "/CN=system:kube-controller-manager" -out kube-controller-manager.csr
  openssl x509 -req -in kube-controller-manager.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000

  echo "------------------------------------------------------------"
  echo "Step 2: Generating KubeConfig for Controller Manager"
  echo "-------------------------------------------------------------"  
  sleep $delay

  cp -v /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin

    kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

    kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig

    echo "------------------------------------------------------------"
    echo "Step 3: Copying the files to kubernetes directory"
    echo "-------------------------------------------------------------"      
    sleep $delay

    cp -v kube-controller-manager.crt kube-controller-manager.key kube-controller-manager.kubeconfig ca.key /var/lib/kubernetes/

    echo "------------------------------------------------------------"
    echo "Step 4: Configuring SystemD service file:"
    echo "-------------------------------------------------------------"  
    sleep $delay

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
--address=0.0.0.0 \\
--service-cluster-ip-range=10.32.0.0/24 \\
--cluster-cidr=10.200.0.0/16 \\
--kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--leader-elect=true \\
--cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \\
--cluster-signing-key-file=/var/lib/kubernetes/ca.key \\
--root-ca-file=/var/lib/kubernetes/ca.crt \\
--service-account-private-key-file=/var/lib/kubernetes/service-account.key \\
--use-service-account-credentials=true \\
--v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

echo "------------------------------------------------------------"
echo "Step 5: Start Controller Manager:"
echo "-------------------------------------------------------------"  
sleep $delay

cp -v /root/binaries/kubernetes/server/bin/kube-controller-manager /usr/local/bin
systemctl start kube-controller-manager
systemctl status kube-controller-manager
systemctl enable kube-controller-manager

SCRIPT

$configure_scheduler = <<-SCRIPT
  node_name=$1
  node_type=$2
  SERVER_IP=$3
  delay=$4

  echo "------------------------------------------------------------"
  echo "Step 1: Generate Certificates for Scheduler"
  echo "-------------------------------------------------------------"
  sleep $delay

  export PATH=$PATH:/usr/local/bin

  cd /root/certificates

  openssl genrsa -out kube-scheduler.key 2048
  openssl req -new -key kube-scheduler.key -subj "/CN=system:kube-scheduler" -out kube-scheduler.csr
  openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-scheduler.crt -days 1000

  echo "------------------------------------------------------------"
  echo "Step 2: Generating KubeConfig for Scheduler"
  echo "-------------------------------------------------------------"  
  sleep $delay

  cp -v /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin

  kubectl config set-cluster kubernetes-from-scratch \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.crt \
  --client-key=kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-from-scratch \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig

echo "------------------------------------------------------------"
echo "Step 3: Copying the files to kubernetes directory"
echo "-------------------------------------------------------------"      
sleep $delay

cp -v kube-scheduler.kubeconfig /var/lib/kubernetes/

echo "------------------------------------------------------------"
echo "Step 4: Configuring SystemD service file for Scheduler:"
echo "-------------------------------------------------------------"  
sleep $delay

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --bind-address=127.0.0.1 \\
  --leader-elect=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

echo "------------------------------------------------------------"
echo "Step 5: Start Scheduler:"
echo "-------------------------------------------------------------"  
sleep $delay

cp -v /root/binaries/kubernetes/server/bin/kube-scheduler /usr/local/bin
systemctl start kube-scheduler
systemctl status kube-scheduler
systemctl enable kube-scheduler

SCRIPT

$configure_adminuser = <<-SCRIPT
  node_name=$1
  node_type=$2
  SERVER_IP=$3
  delay=$4

  echo "------------------------------------------------------------"
  echo "Step 1: Generate Certificates for Admin user"
  echo "-------------------------------------------------------------"
  sleep $delay

  export PATH=$PATH:/usr/local/bin

  cd /root/certificates  

  openssl genrsa -out admin.key 2048
  openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
  openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out admin.crt -days 1000

  echo "------------------------------------------------------------"
  echo "Step 2: Generating KubeConfig for Admin User"
  echo "-------------------------------------------------------------"  
  sleep $delay

  cp -v /root/binaries/kubernetes/server/bin/kubectl /usr/local/bin  

  kubectl config set-cluster kubernetes-from-scratch \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://${SERVER_IP}:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-from-scratch \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig

echo "------------------------------------------------------------"
echo "Step 3: Verify Cluster Status"
echo "-------------------------------------------------------------"  
sleep $delay

kubectl get componentstatuses --kubeconfig=admin.kubeconfig
cp /root/certificates/admin.kubeconfig ~/.kube/config
kubectl get componentstatuses

echo "------------------------------------------------------------"
echo "Step 4: Verify Kubernetes Objects Creation"
echo "-------------------------------------------------------------"  
sleep $delay

kubectl create namespace kplabs
kubectl get namespace kplabs -o yaml
kubectl get serviceaccount --namespace kplabs
kubectl get secret --namespace kplabs
kubectl create serviceaccount demo

SCRIPT

$create_cert_for_worker = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3
WORKER_NODE_IP=$4


echo "------------------------------------------------------------"
echo "Step 1: Generate Certificates for Worker node $WORKER_NODE_IP $WORKER_NODE_NAME"
echo "-------------------------------------------------------------"
sleep $delay

export PATH=$PATH:/usr/local/bin

cd /root/certificates

cat > openssl-kplabs-cka-worker.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${node_name}
IP.1 = ${WORKER_NODE_IP}
EOF

openssl genrsa -out kplabs-cka-worker.key 2048

# TODO: make worker node name configurable
openssl req -new -key kplabs-cka-worker.key -subj "/CN=system:node:kplabs-cka-worker/O=system:nodes" -out kplabs-cka-worker.csr -config openssl-kplabs-cka-worker.cnf
openssl x509 -req -in kplabs-cka-worker.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kplabs-cka-worker.crt -extensions v3_req -extfile openssl-kplabs-cka-worker.cnf -days 1000

openssl genrsa -out kube-proxy.key 2048
openssl req -new -key kube-proxy.key -subj "/CN=system:kube-proxy" -out kube-proxy.csr
openssl x509 -req -in kube-proxy.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -out kube-proxy.crt -days 1000

SCRIPT


$copy_certificates = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3
SERVER_IP=$4
password=$5

echo "------------------------------------------------------------"
echo "Step 0: Copy certificates from master to node1"
echo "-------------------------------------------------------------"
sleep $delay

su - vagrant -c 'cat /dev/zero | ssh-keygen -q -N ""'
su - vagrant -c 'sshpass -p "'${password}'" ssh vagrant@c1-master1 id'
#this command can also copy a public key, 
#but ssh-copy checks if the key is already present
#su - vagrant -c 'sshpass -p "'${password}'" ssh vagrant@"'${SERVER_IP}'" "cat >> /home/vagrant/.ssh/authorized_keys" < /home/vagrant/.ssh/id_rsa.pub'
su - vagrant -c 'sshpass -p "'${password}'" ssh-copy-id "'${SERVER_IP}'"'
su - vagrant -c 'ssh -o StrictHostKeyChecking=no "'${SERVER_IP}'" "sudo rm -rvf /tmp/certificates && mkdir -pv /tmp/certificates && sudo cp -prv /root/certificates/{kube-proxy.crt,kube-proxy.key,kplabs-cka-worker.crt,kplabs-cka-worker.key,ca.crt,ca.key} /tmp/certificates/ && sudo chown vagrant:vagrant -Rv /tmp/certificates"'
su - vagrant -c 'sshpass -p "'${password}'" sudo chown -Rv vagrant:vagrant /tmp/certificates'
rm -rfv /tmp/certificates
su - vagrant -c 'scp -o StrictHostKeyChecking=no -r "'${SERVER_IP}'":/tmp/certificates/ /tmp/'

echo
echo "Transfer of certificates from ${SERVER_IP} to ${node_name} has finished"
echo
sleep $delay

rm -rfv /root/certificates/
mkdir -pv /root/certificates/
cd /tmp/certificates
mv -v kube-proxy.crt kube-proxy.key kplabs-cka-worker.crt kplabs-cka-worker.key ca.crt ca.key /root/certificates
chown -Rv root:root /root/certificates

SCRIPT

$copy_binaries = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3
SERVER_IP=$4
password=$5

echo "------------------------------------------------------------"
echo "Step 0: Copy binaries from master to etcd2"
echo "-------------------------------------------------------------"
sleep $delay

su - vagrant -c 'cat /dev/zero | ssh-keygen -q -N ""'
su - vagrant -c 'sshpass -p "'${password}'" ssh vagrant@c1-master1 id'
#this command can also copy a public key, 
#but ssh-copy checks if the key is already present
#su - vagrant -c 'sshpass -p "'${password}'" ssh vagrant@"'${SERVER_IP}'" "cat >> /home/vagrant/.ssh/authorized_keys" < /home/vagrant/.ssh/id_rsa.pub'
su - vagrant -c 'sshpass -p "'${password}'" ssh-copy-id "'${SERVER_IP}'"'
su - vagrant -c 'ssh -o StrictHostKeyChecking=no "'${SERVER_IP}'" "sudo rm -rvf /tmp/binaries && mkdir -pv /tmp/binaries && sudo cp -prv /root/binaries/ /tmp/ && sudo chown vagrant:vagrant -Rv /tmp/binaries"'
su - vagrant -c 'sshpass -p "'${password}'" sudo chown -Rv vagrant:vagrant /tmp/binaries'
rm -rfv /tmp/binaries
su - vagrant -c 'scp -o StrictHostKeyChecking=no -r "'${SERVER_IP}'":/tmp/binaries /tmp/'

echo
echo "Transfer of binaries from ${SERVER_IP} to ${node_name} has finished"
echo
sleep $delay

rm -rfv /root/binaries/
mv -v /tmp/binaries/ /root/
chown -Rv root:root /root/binaries

SCRIPT

$configure_worker_node = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3
SERVER_IP=$4

export PATH=$PATH:/usr/local/bin

echo "------------------------------------------------------------"
echo "Pre-Requisites:"
echo "-------------------------------------------------------------"
sleep $delay

setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum -y install docker && systemctl start docker && systemctl enable docker
yum -y install socat conntrack ipset
sysctl -w net.ipv4.conf.all.forwarding=1
cd  /root/binaries/kubernetes/node/bin/
cp kube-proxy kubectl kubelet /usr/local/bin

echo "------------------------------------------------------------"
echo "Step 4: Move Certificates to Specific Location."
echo "-------------------------------------------------------------"
sleep $delay

cd /root/certificates
mkdir -pv /var/lib/kubernetes
cp -v ca.crt /var/lib/kubernetes
mkdir -pv /var/lib/kubelet
cp -v kplabs-cka-worker.crt  kplabs-cka-worker.key  kube-proxy.crt  kube-proxy.key /var/lib/kubelet/

echo "------------------------------------------------------------"
echo "Step 5: Generate Kubelet Configuration YAML File:"
echo "-------------------------------------------------------------"
sleep $delay

cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
EOF

echo "------------------------------------------------------------"
echo "Step 6: Generate Systemd service file for kubelet:"
echo "-------------------------------------------------------------"
sleep $delay

cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --tls-cert-file=/var/lib/kubelet/kplabs-cka-worker.crt \\
  --tls-private-key-file=/var/lib/kubelet/kplabs-cka-worker.key \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2 \\
  --cgroup-driver=systemd \\
  --runtime-cgroups=/systemd/system.slice \\
  --kubelet-cgroups=/systemd/system.slice
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

echo "------------------------------------------------------------"
echo "Step 7: Generate the Kubeconfig file for Kubelet"
echo "-------------------------------------------------------------"
sleep $delay

cd /var/lib/kubelet
cp /var/lib/kubernetes/ca.crt .
SERVER_IP=$SERVER_IP

kubectl config set-cluster kubernetes-from-scratch \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://${SERVER_IP}:6443 \
    --kubeconfig=kplabs-cka-worker.kubeconfig

  kubectl config set-credentials system:node:kplabs-cka-worker \
    --client-certificate=kplabs-cka-worker.crt \
    --client-key=kplabs-cka-worker.key \
    --embed-certs=true \
    --kubeconfig=kplabs-cka-worker.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-from-scratch \
    --user=system:node:kplabs-cka-worker \
    --kubeconfig=kplabs-cka-worker.kubeconfig

  kubectl config use-context default --kubeconfig=kplabs-cka-worker.kubeconfig

  mv -v kplabs-cka-worker.kubeconfig kubeconfig

  echo "------------------------------------------------------------"
  echo "Kube-Proxy configuration"
  echo "-------------------------------------------------------------"
  sleep $delay

  echo "------------------------------------------------------------"
  echo "Step : copy kube-proxy certificate to /var/lib/kube-proxy dir"
  echo "-------------------------------------------------------------"
  sleep $delay

  mkdir -pv /var/lib/kube-proxy
  #cp -v /root/certificates/kube-proxy.crt /var/lib/kubelet/ 

  echo "------------------------------------------------------------"
  echo "Step 2: Generate the Kubeconfig file for Kube-proxy"
  echo "-------------------------------------------------------------"
  sleep $delay
  cd /var/lib/kube-proxy
  cp -pv /root/certificates/* /var/lib/kube-proxy/
  #cp -pv /root/certificates/kube-proxy.crt /var/lib/kube-proxy

  SERVER_IP=$SERVER_IP

  kubectl config set-cluster kubernetes-from-scratch \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --server=https://${SERVER_IP}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.crt \
  --client-key=kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-from-scratch \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

mv -v kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

echo "------------------------------------------------------------"
echo "Step 3: Generate kube-proxy configuration file:"
echo "-------------------------------------------------------------"
sleep $delay
cd /var/lib/kube-proxy

cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

echo "------------------------------------------------------------"
echo "Step 4: Create kube-proxy service file:"
echo "-------------------------------------------------------------"
sleep $delay

cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

echo "------------------------------------------------------------"
echo "Step 5: start kubelet and kube-proxy services"
echo "-------------------------------------------------------------"
sleep $delay

systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy

SCRIPT

$configure_networking_worker = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3

echo "------------------------------------------------------------"
echo "Step 1: Download CNI Plugins:"
echo "-------------------------------------------------------------"
sleep $delay

cd /tmp
wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz

echo "------------------------------------------------------------"
echo "Step 2: Configure Base Directories:"
echo "-------------------------------------------------------------"
sleep $delay

mkdir -pv \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/run/kubernetes

  echo "------------------------------------------------------------"
  echo "Step 3: Move CNI Tar File And Extract it."
  echo "-------------------------------------------------------------"
  sleep $delay

  mv -v cni-plugins-linux-amd64-v0.8.6.tgz /opt/cni/bin
  cd /opt/cni/bin
  tar -xzvf cni-plugins-linux-amd64-v0.8.6.tgz


SCRIPT

$configure_networking_master = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3

echo "------------------------------------------------------------"
echo "Step 4: Configuring Weave (Run this step on Master Node)"
echo "-------------------------------------------------------------"
sleep $delay

sudo -u vagrant /usr/local/bin/kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(/usr/local/bin/kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.200.0.0/16"

SCRIPT

$configure_rbac = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3

echo "------------------------------------------------------------"
echo "Step : Configuring RBAC (Run this step on Master Node)"
echo "-------------------------------------------------------------"
sleep $delay

cat <<EOF | sudo -u vagrant /usr/local/bin/kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | sudo -u vagrant /usr/local/bin/kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF

SCRIPT

$configure_dns = <<-SCRIPT
node_name=$1
node_type=$2
delay=$3

echo "------------------------------------------------------------"
echo "Step : Configuring DNS (based on CoreDNS)"
echo "-------------------------------------------------------------"
sleep $delay

mkdir -pv /root/yamls
cd /root/yamls

wget https://raw.githubusercontent.com/zealvora/certified-kubernetes-administrator/master/Domain%206%20-%20Cluster%20Architecture%2C%20Installation%20%26%20Configuration/coredns.yaml

sudo -u vagrant /usr/local/bin/kubectl apply -f coredns.yaml

SCRIPT



Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "n1" do |n1|
    n1.vm.box = "bento/centos-7.6"
    n1.vm.hostname = "kplabs-cka-worker"
    n1.vm.network :private_network, ip: "172.16.94.11"
	
        config.vm.provision "misc", type: "shell" do |shell|
          shell.inline = $install_misc
          shell.args = ["c1-node1","worker"]
        end

        config.vm.provision "configure_hosts", type: "shell" do |shell|
          shell.inline = $configure_hosts
          shell.args = ["c1-node1","worker"]
        end

        config.vm.provision "download_binaries_node", type: "shell" do |shell|
          shell.inline = $download_binaries_node
          shell.args = ["c1-node1","worker"]
        end

        config.vm.provision "disable_swap", type: "shell" do |shell|
          shell.inline = $disable_swap
          shell.args = ["c1-node1","worker"]
        end

        config.vm.provision "copy_certificates", type: "shell" do |shell|
          shell.inline = $copy_certificates
          shell.args = ["c1-node1","worker","3", "172.16.94.10","vagrant"]
        end

        config.vm.provision "configure_worker_node", type: "shell" do |shell|
          shell.inline = $configure_worker_node
          shell.args = ["c1-node1","worker","5", "172.16.94.10"]
        end        

        config.vm.provision "configure_networking_worker", type: "shell" do |shell|
          shell.inline = $configure_networking_worker
          shell.args = ["c1-node1","worker","5"]
        end
  end

  config.vm.define "n2" do |n2|
    n2.vm.box = "bento/centos-7.6"
    n2.vm.hostname = "c1-etcd2"
    n2.vm.network :private_network, ip: "172.16.94.12"

        config.vm.provision "misc", type: "shell" do |shell|
          shell.inline = $install_misc
          shell.args = ["c1-etcd2","worker"]
        end

        config.vm.provision "configure_hosts", type: "shell" do |shell|
          shell.inline = $configure_hosts
          shell.args = ["c1-etcd2","worker"]
        end

        config.vm.provision "download_binaries_srv", type: "shell" do |shell|
          shell.inline = $download_binaries_srv
          shell.args = ["c1-master1","master"]
        end        

        config.vm.provision "copy_binaries", type: "shell" do |shell|
          shell.inline = $copy_binaries
          shell.args = ["c1-etcd2","worker","3", "172.16.94.10","vagrant"]
        end        
        
        config.vm.provision "copy_certificates", type: "shell" do |shell|
          shell.inline = $copy_certificates
          shell.args = ["c1-etcd2","worker","3", "172.16.94.10","vagrant"]
        end
        
        config.vm.provision "configure_etcd", type: "shell" do |shell|
          shell.inline = $configure_etcd
          shell.args = ["c1-etcd2","worker","172.16.94.12"]
        end

        config.vm.provision "restore_etcd", type: "shell" do |shell|
          shell.inline = $restore_etcd
          shell.args = ["c1-etcd2","worker","172.16.94.12","172.16.94.10"]
        end
        
  end

  config.vm.define "km1" do |km1|
    km1.vm.box = "bento/centos-7.6"
    km1.vm.hostname = "c1-master1"
    km1.vm.network :private_network, ip: "172.16.94.10"
	  km1.vm.network "forwarded_port", guest: 8001, host: 8888

        config.vm.provision "misc", type: "shell" do |shell|
          shell.inline = $install_misc
          shell.args = ["c1-master1","master"]
        end

        config.vm.provision "configure_hosts", type: "shell" do |shell|
            shell.inline = $configure_hosts
            shell.args = ["c1-master1","master"]
        end

        config.vm.provision "download_binaries_srv", type: "shell" do |shell|
          shell.inline = $download_binaries_srv
          shell.args = ["c1-master1","master"]
        end

        config.vm.provision "create_ca_cert", type: "shell" do |shell|
          shell.inline = $create_ca_cert
          shell.args = ["c1-master1","master"]
        end

        config.vm.provision "configure_etcd", type: "shell" do |shell|
          shell.inline = $configure_etcd
          shell.args = ["c1-master1","master","172.16.94.10"]
        end
        
        config.vm.provision "configure_api_server", type: "shell" do |shell|
          shell.inline = $configure_api_server
          shell.args = ["c1-master1","master","172.16.94.10"]
        end

        config.vm.provision "configure_controller_manager", type: "shell" do |shell|
          shell.inline = $configure_controller_manager
          shell.args = ["c1-master1","master","172.16.94.10","3"]
        end

        config.vm.provision "configure_scheduler", type: "shell" do |shell|
          shell.inline = $configure_scheduler
          shell.args = ["c1-master1","master","172.16.94.10","3"]
        end

        config.vm.provision "configure_adminuser", type: "shell" do |shell|
          shell.inline = $configure_adminuser
          shell.args = ["c1-master1","master","172.16.94.10","3"]
        end

        config.vm.provision "create_cert_for_worker", type: "shell" do |shell|
          shell.inline = $create_cert_for_worker
          shell.args = ["kplabs-cka-worker","master", "3","172.16.94.11"]
        end

        config.vm.provision "configure_networking_master", type: "shell" do |shell|
          shell.inline = $configure_networking_master
          shell.args = ["c1-master1","worker","5"]
        end

        config.vm.provision "configure_rbac", type: "shell" do |shell|
          shell.inline = $configure_rbac
          shell.args = ["c1-master1","master","5"]
        end

        config.vm.provision "configure_dns", type: "shell" do |shell|
          shell.inline = $configure_dns
          shell.args = ["c1-master1","master","5"]
        end

        config.vm.provision "backup_etcd", type: "shell" do |shell|
          shell.inline = $backup_etcd
          shell.args = ["c1-master1","master","5"]
        end
        
  end
   
  config.vm.provider :virtualbox do |vb|
    vb.customize [ "modifyvm", :id, "--memory", 1024 * 4 ]
    vb.customize [ "modifyvm", :id, "--cpus", "2" ]
  end
end
