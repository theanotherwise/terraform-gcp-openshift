# OpenShift 3.9 - 3.11

## Terraform management
```bash
terraform apply -var-file=variables.tfvars

terraform plan -var-file=variables.tfvars

terraform plan -var-file=variables.tfvars
```

## OpenShift repository
```
git clone https://github.com/openshift/openshift-ansible.git

git branch -a

git checkout release-3.11 # for 3.11
# git checkout release-3.9 # for 3.9
```

## Node names
```bash
openshift-compute-0
openshift-compute-1
..
openshift-compute-n

openshift-infra-0
openshift-infra-1
(...)
openshift-infra-n

openshift-master-0
openshift-master-1
(...)
openshift-master-n
```

## Access to Openshift
```bash
ssh-add terraform.id_rsa

ssh -A -i terraform.id_rsa terraform@34.66.62.74
```

## Disable `systemd-resolved` on Bastion
```bash
sudo -i systemctl disable systemd-resolved
sudo -i systemctl stop systemd-resolved

sudo -i rm /etc/resolv.conf

sudo -i tee /etc/resolv.conf <<"EndOfMessage"
search local
nameserver 169.254.169.254
EndOfMessage
```

## Access to `OpenShift` nodes
```bash
ssh ansible@openshift-XXX-n.local
```

## Deploy ssh keys if required
```bash
ssh-keygen

HOST_LIST="openshift-compute-0 openshift-compute-1 openshift-infra-0 openshift-master-0 openshift-master-1"

for i in $HOST_LIST ; do 
  ssh-copy-id -i ~/.ssh/id_rsa.pub $i;
done
```

## Install ansible 

### Openshift `3.9` on CentOS 7
```
yum install ansible 
```

### Openshift `3.11` on Ubuntu 18 LTS from `apt`
```bash
apt-get update
apt-get install -y ansible 
```

### Openshift `3.11` on Ubuntu 18 LTS from `pip` - latest version
```bash
apt-get update
apt-get install -y python3-pip

pip3 install --upgrade pip

pip3 install ansible
```

## Prepare network on nodes (just on first run)
```bash
sudo -i rm -f /etc/resolv.conf
sudo -i rm -f /etc/NetworkManager/NetworkManager.conf

sudo -i tee /etc/resolv.conf <<"EndOfMessage"
search local
nameserver 169.254.169.254
EndOfMessage

sudo -i tee /etc/NetworkManager/NetworkManager.conf <<"EndOfMessage"
[main]
dns=none
[logging]
EndOfMessage

sudo -i systemctl restart NetworkManager
sudo -i systemctl restart network

clear

cat /etc/NetworkManager/NetworkManager.conf
cat /etc/resolv.conf
```

## Ansible `inventory.ini` for Openshift `3.11`
```bash
cat > inventory.ini << "EndOfMessage"
[OSEv3:children]
masters
nodes
etcd
lb

[masters]
openshift-master-0.local

[lb]
openshift-lb-0.local

[etcd]
openshift-master-0.local

[nodes]
openshift-master-0.local  openshift_node_group_name='node-config-master'
openshift-compute-0.local openshift_node_group_name='node-config-compute'
openshift-infra-0.local   openshift_node_group_name='node-config-infra'

[nodes:vars]
openshift_disable_check=disk_availability,memory_availability,docker_storage

[masters:vars]
openshift_disable_check=disk_availability,memory_availability,docker_storage

[OSEv3:vars]
ansible_ssh_user=ansible
openshift_deployment_type=origin
ansible_become=true
openshift_release=v3.11
openshift_master_default_subdomain=apps.local
openshift_master_cluster_hostname=openshift-master-0.local
debug_level=2
EndOfMessage
```

## Ansible `inventory.ini` for Openshift `3.9`
```bash
[OSEv3:children]
masters
nodes
etcd
lb

[masters]
openshift-master-0.local

[lb]
openshift-lb-0.local

[etcd]
openshift-master-0.local

[nodes]
openshift-master-0.local
openshift-infra-0.local       openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
openshift-compute-0.local     openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

[nodes:vars]
openshift_disable_check=disk_availability,memory_availability,docker_storage

[masters:vars]
openshift_disable_check=disk_availability,memory_availability,docker_storage

[OSEv3:vars]
ansible_user=ansible
openshift_deployment_type=origin
ansible_become=true
openshift_release=v3.9
openshift_master_default_subdomain=apps.local
openshift_master_cluster_hostname=openshift-master-0.local
debug_level=2
template_service_broker_selector={'region': 'infra'}
```

#### If required ommit some checks
```bash
openshift_check_min_host_disk_gb=1 openshift_check_min_host_memory_gb=1
```

## OpenShift ansible playbooks

### Install
```bash
ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml
ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml
```

### Uninstall
```bash
ansible-playbook -i inventory.ini openshift-ansible/playbooks/adhoc/uninstall.yml
```

### Last steps
```bash
htpasswd -c /etc/origin/master/htpasswd admin
```

## Diagnostics

### Get pods which status is not Running
```bash
for i in `oc get ns | tail -n +2 | awk '{print $1}'` ; do 
  echo "Namespace: $i"
  oc get pods -n $i 2>/dev/null | tail -n +2 | awk '$3!="Running" {print $1,$3}' 
  echo
done
```

### Get resources
```bash
NAMESPACES="default example"
RESOURCES="pods pvc pv sc cm ingress deployments svc"

for i in $NAMESPACES ; do
  echo "Namespace: $i"
  for j in $RESOURCES ; do
    oc get "$j" -n "$i" 
  done
  echo
done
```

### login as admin
```bash
oc login -u system:admin -n default
```

### Pushing docker images
```
# oc get svc/docker-registry -n default

OS_REG_ADDR="172.30.13.74"
OS_REG_PORT="5000"

sudo -i tee /etc/docker/daemon.json <<EndOfMessage
{
        "insecure-registries" : ["$OS_REG_ADDR:$OS_REG_PORT"]
}
EndOfMessage

sudo -i service docker restart
sudo -i chmod 777 /var/run/docker.sock

DOCKER_URL="docker.io"
DOCKER_VENDOR="grafana"
DOCKER_IMAGE="grafana"
test -z $DOCKER_VENDOR && DOCKER_PATH="$DOCKER_IMAGE" || DOCKER_PATH="$DOCKER_VENDOR/$DOCKER_IMAGE"
DOCKER_VERSION="latest"
DOCKER_PATH="$DOCKER_URL/$DOCKER_PATH"

OS_PROJ_NAME="linuxpolska"
OS_IMAGE_PATH="$OS_PROJ_NAME/$DOCKER_IMAGE"

oc project "$OS_PROJ_NAME"

oc login
OC_TOKEN="`oc whoami -t`"

oc login -u system:admin -n default

docker login -u admin -p "$OC_TOKEN" "$OS_REG_ADDR:$OS_REG_PORT"

docker pull "$DOCKER_PATH:$DOCKER_VERSION"

docker tag "$DOCKER_PATH:$DOCKER_VERSION" "$OS_REG_ADDR:$OS_REG_PORT/$OS_IMAGE_PATH"
docker push "$OS_REG_ADDR:$OS_REG_PORT/$OS_IMAGE_PATH"
```

### Network diagnostics
```bash
nsenter -t PID -n ip addr

conntrack -L
```

## Certificates 

### Pods not starting on `OpenShift 3.9`

```bash
openshift-master/redeploy-openshift-ca.yml
```

### Fix certificates on x509 Error, deploy whole `in order`
```bash
redeploy-certificates.yml

openshift-etcd/redeploy-ca.yml

openshift-master/redeploy-certificates.yml
openshift-master/redeploy-openshift-ca.yml
```

### Other certificates
```bash
openshift-node/redeploy-certificates.yml

openshift-hosted/redeploy-registry-certificates.yml
openshift-hosted/redeploy-router-certificates.yml
```

# Local Persistent Volume
```bash
LPV_ALIAS="grafana0"
LPV_DIR=`mktemp -d $LPV_ALIAS-XXXXXXXXXX`

mkdir "$LPV_DIR/pvc"
mkdir "$LPV_DIR/pv"

cat > $LPV_DIR/pvc-template.yml << EndOfMesage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: $LPV_ALIAS-pvc-XXXXX
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: $LPV_ALIAS-sc-XXXXX
EndOfMesage

cat > $LPV_DIR/pv-template.yml << EndOfMesage
apiVersion: v1
kind: PersistentVolume
metadata:
  name: $LPV_ALIAS-pv-XXXXX
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: $LPV_ALIAS-sc-XXXXX
  local:
    path: /mnt/local-storage
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - openshift-compute-0
EndOfMesage

for i in `seq -w 0 20` ; do 
  cat $LPV_DIR/pvc-template.yml | sed "s/XXXXX/$i/g " > $LPV_DIR/pvc/$i.yml 
  cat $LPV_DIR/pv-template.yml | sed "s/XXXXX/$i/g " > $LPV_DIR/pv/$i.yml
done

for i in `ls "$LPV_DIR/pv" | xargs -0` ; do oc create -f ./$LPV_DIR/pv/$i ; done
for i in `ls "$LPV_DIR/pvc" | xargs -0` ; do oc create -f ./$LPV_DIR/pvc/$i ; done
```

# Self signed for web access

```bash
openssl req -newkey rsa:4096 \
            -x509 \
            -sha256 \
            -days 3650 \
            -nodes \
            -out server.crt \
            -keyout server.key
```
