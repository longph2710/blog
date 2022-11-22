1. install dependencies
```
sudo apt install python3-dev libffi-dev gcc libssl-dev

# python pip
sudo apt install python3-pip

# ensure the lasted version of pip is installed
sudo pip3 install -U pip

# install ansible
sudo pip install -U 'ansible>=4,<6'
```

2. install kolla ansible for development

```
# clone yoga release
git clone --branch stable/yoga https://opendev.org/openstack/kolla-ansible

# install kolla ansible
sudo pip3 install ./kolla-ansible

# create /etc/kolla dir
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# copy conf file to /etc/kolla
cp -r kolla-ansible/etc/kolla/* /etc/kolla

# copy inventory file to current dir
cp kolla-ansible/ansible/inventory/* .
```

3. install ansible  galaxy requirements

```
kolla-ansible install-deps
```

4. config ansible

add the following options to conf file /etc/ansible/ansible.cfg
```
[default]
host_key_checking=False
pipelining=True
forks=100
```

5. config globals.yml

```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"

network_interface: enp0s3
neutron_external_interface: enp0s8
kolla_internal_vip_address: 10.0.2.15

nova_compute_virt_type: "qemu"

enable_haproxy: "no"

# cinder
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_lvm: "yes"

# swift
enable_swift: "yes"
enable_swift_s3api: "yes"

# trove
enable_trove: "yes"
```

6. create diskspace partitiion for cinder

```
apt instal lvm2
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb
```

7. config swift

```
sudo apt install xfsprogs

sudo parted /dev/sdc -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
sudo parted /dev/sdd -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1
sudo parted /dev/sde -s -- mklabel gpt mkpart KOLLA_SWIFT_DATA 1 -1

sudo mkfs.xfs -f -L d0 /dev/sdc1
sudo mkfs.xfs -f -L d0 /dev/sdd1
sudo mkfs.xfs -f -L d0 /dev/sde1

sudo nano init-swift-ring
# add following content
---
#!/bin/bash
export KOLLA_INTERNAL_ADDRESS=10.10.20.33
export KOLLA_SWIFT_BASE_IMAGE="kolla/ubuntu-source-swift-base:4.0.0"
mkdir -p /etc/kolla/config/swift
# Object ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/object.builder create 10 3 1
for i in {0..2}; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/object.builder add r1z1-${KOLLA_INTERNAL_ADDRESS}:6000/d${i} 1;
done
# Account ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/account.builder create 10 3 1
for i in {0..2}; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/account.builder add r1z1-${KOLLA_INTERNAL_ADDRESS}:6001/d${i} 1;
done
# Container ring
docker run \
  --rm \
  -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
  $KOLLA_SWIFT_BASE_IMAGE \
  swift-ring-builder \
    /etc/kolla/config/swift/container.builder create 10 3 1
for i in {0..2}; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      etc/kolla/config/swift/container.builder add r1z1-${KOLLA_INTERNAL_ADDRESS}:6002/d${i} 1;
done
for ring in object account container; do
  docker run \
    --rm \
    -v /etc/kolla/config/swift/:/etc/kolla/config/swift/ \
    $KOLLA_SWIFT_BASE_IMAGE \
    swift-ring-builder \
      /etc/kolla/config/swift/${ring}.builder rebalance;
done
---

sudo chmod 755 init-swift-ring
sudo ./init-swift-ring
```

8. deployment

```
bootstrap-servers
prechecks
pull
deploy
post-deploy
```

9. FINAL
genpassword
PASS: 29IC6r5FNCNGC9lCQwqjo6yAM49aqKrPL9H0E6Ic