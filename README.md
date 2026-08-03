# Lab OpenStack All-in-One với Kolla Ansible

Đây là dự án phòng lab OpenStack cá nhân được triển khai bằng **Kolla Ansible** trên máy ảo Ubuntu Server chạy trong VirtualBox. Dự án mô phỏng đầy đủ vòng đời của một môi trường OpenStack, bao gồm triển khai hệ thống, cấu hình mạng, quản lý image, tạo máy ảo, kiểm tra tính bền vững của block storage, orchestration, quản lý danh tính và theo dõi tài nguyên.

> Repository này được xây dựng cho mục đích học tập, assessment và thực hành troubleshooting. \
## Tổng quan dự án

| Hạng mục | Giá trị |
|---|---|
| Mô hình triển khai | OpenStack All-in-One |
| Phiên bản OpenStack | 2026.1 |
| Kolla Ansible | 22.0.0 |
| Hệ điều hành host | Ubuntu Server 24.04.4 LTS |
| Nền tảng ảo hóa | Oracle VirtualBox |
| Máy vật lý | Windows 11 Home |
| Hostname OpenStack | `openstack-aio` |
| Hypervisor | QEMU/libvirt |
| Container runtime | Docker |
| Inventory triển khai | `inventory/all-in-one` |
| Cấu hình Kolla chính | `config/globals.yml` |

## Tài nguyên phòng lab

| Tài nguyên | Cấu hình |
|---|---:|
| vCPU | 8 |
| RAM | 16 GB |
| Disk hệ điều hành | 70 GB |
| Root filesystem sau khi mở rộng | Khoảng 67 GB |
| Disk Cinder | 30 GB |
| Số network interface | 3 |
| Cinder backend | LVM |
| Cinder volume group | `cinder-volumes` |

Cấu hình tối thiểu chính thức của Kolla Ansible chỉ phù hợp với môi trường đánh giá rất nhỏ. Một phòng lab thực tế nên có nhiều CPU, RAM và dung lượng disk hơn mức tối thiểu. Môi trường này sử dụng ba network interface, 16 GB RAM và hai disk riêng cho hệ điều hành và Cinder.

## Thiết kế mạng

| Interface | Chế độ VirtualBox | Địa chỉ | Mục đích |
|---|---|---|---|
| `enp0s3` | NAT | `10.0.2.15/24` | Truy cập Internet để tải package và container image |
| `enp0s8` | Host-only | `192.168.56.10/24` | Management, SSH và OpenStack API |
| `enp0s9` | Host-only | Không cấu hình IPv4 | Provider/external network của Neutron |

| Thành phần mạng | Giá trị |
|---|---|
| Kolla internal VIP | `192.168.56.20` |
| Management network | `192.168.56.0/24` |
| Provider network | `192.168.100.0/24` |
| Provider gateway | `192.168.100.1` |
| Floating IP pool | `192.168.100.100-192.168.100.200` |
| Private network | `10.10.10.0/24` |
| Private gateway | `10.10.10.1` |

External interface phải ở trạng thái hoạt động nhưng thông thường không mang địa chỉ IP vì Neutron sẽ gắn interface này vào external bridge. VIP phải là địa chỉ chưa được sử dụng, thuộc management network và có thể được Keepalived quản lý.

## Kiến trúc tổng thể

```text
Windows 11 Host
│
└── VirtualBox
    │
    └── Ubuntu Server: openstack-aio
        ├── Môi trường triển khai Kolla Ansible
        ├── Controller services
        ├── Network services
        ├── Nova Compute với QEMU/libvirt
        ├── Cinder LVM backend
        └── Docker containers

Windows management network
        │
        ├── 192.168.56.10  Ubuntu management IP
        └── 192.168.56.20  Kolla API/Horizon VIP

Provider network 192.168.100.0/24
        │
        ├── 192.168.100.1   Windows provider gateway
        ├── 192.168.100.101 Neutron router gateway
        └── 192.168.100.166 Floating IP của test-vm

Private network 10.10.10.0/24
        │
        └── test-vm: 10.10.10.59
```

## Các service đã bật

| Nhóm | Service | Trạng thái |
|---|---|---|
| Identity | Keystone | Đã kiểm thử |
| Image | Glance | Đã kiểm thử |
| Compute | Nova | Đã kiểm thử |
| Theo dõi tài nguyên | Placement | Đã kiểm thử |
| Networking | Neutron | Đã kiểm thử |
| Block storage | Cinder | Đã kiểm thử |
| Dashboard | Horizon | Đã kiểm thử |
| Orchestration | Heat | Đã kiểm thử |
| Database | MariaDB | Healthy |
| Message queue | RabbitMQ | Healthy |
| API proxy | HAProxy | Healthy |
| Quản lý VIP | Keepalived | Running |
| Database proxy | ProxySQL | Healthy |
| Virtual switching | Open vSwitch | Healthy |

Swift, Manila, Mistral, Octavia và Barbican chưa được triển khai trong môi trường hiện tại.

## Cấu trúc repository

```text
openstack-kolla-lab/
├── README.md
├── .gitignore
├── config/
│   └── globals.yml
├── heat/
│   └── heat-test.yaml
├── inventory/
│   └── all-in-one
└── templates/
    └── check_alive_proxysql.sh.j2
```

## Cấu hình Kolla chính

```yaml
---
kolla_base_distro: "ubuntu"

network_interface: "enp0s8"
neutron_external_interface: "enp0s9"
kolla_internal_vip_address: "192.168.56.20"

enable_haproxy: "yes"
enable_keepalived: "yes"

enable_keystone: "yes"
enable_glance: "yes"
enable_nova: "yes"
enable_placement: "yes"
enable_neutron: "yes"
enable_horizon: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"

nova_compute_virt_type: "qemu"
```

QEMU được sử dụng vì VirtualBox không cung cấp nested KVM phù hợp trong môi trường này. Trên host có `/dev/kvm`, có thể đặt `nova_compute_virt_type` thành `kvm`.

## Chuẩn bị host

Cài đặt các package cần thiết:

```bash
sudo apt update
sudo apt install -y \
  git \
  python3-dev \
  python3-venv \
  libffi-dev \
  gcc \
  libssl-dev \
  libdbus-glib-1-dev
```

Tạo Python virtual environment:

```bash
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate
pip install --upgrade pip
```

Cài đặt Kolla Ansible:

```bash
pip install \
  git+https://opendev.org/openstack/kolla-ansible@stable/2026.1
```

Cài các Ansible collection cần thiết:

```bash
kolla-ansible install-deps
```

Sao chép cấu hình mẫu và inventory:

```bash
sudo mkdir -p /etc/kolla
sudo chown "$USER:$USER" /etc/kolla

cp -r \
  ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* \
  /etc/kolla/

cp \
  ~/kolla-venv/share/kolla-ansible/ansible/inventory/all-in-one \
  ~/all-in-one
```

## Chuẩn bị Cinder LVM

> Cảnh báo: các lệnh sau sẽ xóa dữ liệu hiện có trên `/dev/sdb`.

```bash
sudo lsblk
sudo pvcreate /dev/sdb
sudo vgcreate cinder-volumes /dev/sdb
sudo pvs
sudo vgs
sudo lvs
```

Cấu hình cần thiết:

```yaml
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
```

Kolla yêu cầu volume group có tên `cinder-volumes` khi sử dụng LVM backend mặc định.

## Quy trình triển khai

Sinh password:

```bash
kolla-genpwd
```

Chạy lần lượt các bước triển khai:

```bash
kolla-ansible bootstrap-servers -i ~/all-in-one
kolla-ansible prechecks -i ~/all-in-one
kolla-ansible pull -i ~/all-in-one
kolla-ansible deploy -i ~/all-in-one
kolla-ansible post-deploy
```

Trình tự thông thường là bootstrap, prechecks, deploy và post-deploy. Việc pull container image riêng giúp tách lỗi registry hoặc network khỏi lỗi trong quá trình deploy.

## OpenStack CLI

Cài CLI trong cùng virtual environment:

```bash
source ~/kolla-venv/bin/activate

pip install python-openstackclient \
  -c https://releases.openstack.org/constraints/upper/2026.1
```

Các plugin bổ sung được dùng trong lab:

```bash
pip install python-heatclient osc-placement
```

Kolla sinh file `/etc/kolla/clouds.yaml` sau bước `post-deploy`. Có thể sao chép vào thư mục cấu hình của user:

```bash
mkdir -p ~/.config/openstack
cp /etc/kolla/clouds.yaml ~/.config/openstack/clouds.yaml
chmod 600 ~/.config/openstack/clouds.yaml
```

Kiểm tra xác thực:

```bash
openstack token issue
```

## Các lệnh kiểm tra hệ thống

```bash
openstack token issue
openstack service list
openstack endpoint list
openstack compute service list
openstack hypervisor list
openstack network agent list
openstack image list
openstack volume service list
sudo docker ps
sudo docker ps -a
```

## Tài nguyên kiểm thử

| Tài nguyên | Giá trị |
|---|---|
| Image | CirrOS 0.6.3 |
| Flavor | `m1.small` |
| vCPU | 1 |
| RAM | 1024 MB |
| Root disk | 5 GB |
| Private network | `private-net` |
| Private subnet | `private-subnet` |
| Provider network | `public-net` |
| Router | `lab-router` |
| Instance | `test-vm` |
| Fixed IP | `10.10.10.59` |
| Floating IP | `192.168.100.166` |
| Key pair | `lab-key` |
| Cinder volume | `data-volume` |
| Dung lượng volume | 1 GB |

## Tạo môi trường kiểm thử

Tạo private network:

```bash
openstack network create private-net

openstack subnet create private-subnet \
  --network private-net \
  --subnet-range 10.10.10.0/24 \
  --gateway 10.10.10.1 \
  --dns-nameserver 8.8.8.8
```

Tạo external provider network:

```bash
openstack network create public-net \
  --external \
  --provider-network-type flat \
  --provider-physical-network physnet1

openstack subnet create public-subnet \
  --network public-net \
  --subnet-range 192.168.100.0/24 \
  --allocation-pool start=192.168.100.100,end=192.168.100.200 \
  --gateway 192.168.100.1 \
  --no-dhcp
```

Tạo router:

```bash
openstack router create lab-router
openstack router add subnet lab-router private-subnet
openstack router set lab-router --external-gateway public-net
```

Cho phép ping và SSH:

```bash
openstack security group rule create --protocol icmp default
openstack security group rule create \
  --protocol tcp \
  --dst-port 22 \
  default
```

Tạo key pair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/openstack_lab

openstack keypair create \
  --public-key ~/.ssh/openstack_lab.pub \
  lab-key
```

Tạo instance:

```bash
openstack server create test-vm \
  --image cirros \
  --flavor m1.small \
  --network private-net \
  --key-name lab-key
```

Gán Floating IP:

```bash
openstack floating ip create public-net
openstack server add floating ip test-vm 192.168.100.166
```

Kết nối từ Windows:

```powershell
ssh -i C:\Users\ADMIN\.ssh\openstack_lab cirros@192.168.100.166
```

## Kiểm thử tính bền vững của Cinder

Tạo và attach volume:

```bash
openstack volume create --size 1 data-volume
openstack server add volume test-vm data-volume
openstack volume list
```

Bên trong instance:

```bash
lsblk
sudo mkfs.ext4 /dev/vdb
sudo mkdir -p /mnt/data
sudo mount /dev/vdb /mnt/data
echo "OpenStack Cinder persistent test" | sudo tee /mnt/data/test.txt
sudo cat /mnt/data/test.txt
```

Volume được detach, attach lại, mount lại và vẫn đọc được file cũ. Kết quả này xác nhận dữ liệu tồn tại độc lập với root disk của instance.

## Kiểm thử Heat Orchestration

File `heat/heat-test.yaml` tạo một Neutron network và subnet.

Kiểm tra template:

```bash
openstack orchestration template validate \
  -t heat/heat-test.yaml
```

Tạo stack:

```bash
openstack stack create \
  -t heat/heat-test.yaml \
  heat-test-stack
```

Kiểm tra stack và các resource:

```bash
openstack stack show heat-test-stack
openstack stack resource list heat-test-stack
openstack network show heat-test-net
openstack subnet show heat-test-subnet
```

Kết quả quan sát được:

| Resource | Loại | Trạng thái |
|---|---|---|
| `heat_test_network` | `OS::Neutron::Net` | `CREATE_COMPLETE` |
| `heat_test_subnet` | `OS::Neutron::Subnet` | `CREATE_COMPLETE` |

## Kiểm thử Placement

Liệt kê resource provider:

```bash
openstack resource provider list
```

Xem inventory:

```bash
openstack resource provider inventory list \
  5370ac7e-6b1a-49ea-8c62-bce5e57a7981
```

Allocation ghi nhận cho `test-vm`:

| Tài nguyên | Đã cấp phát |
|---|---:|
| VCPU | 1 |
| MEMORY_MB | 1024 |
| DISK_GB | 5 |

Các giá trị khớp với flavor `m1.small`, xác nhận Nova và Placement đã ghi nhận đúng tài nguyên của instance.

## Kiểm thử Keystone

Luồng quản lý danh tính được kiểm tra bằng cách tạo project, user, gán role và cấp project-scoped token.

```bash
openstack project create assessment-project
openstack user create --domain default --password-prompt assessment-user

openstack role add \
  --project assessment-project \
  --user assessment-user \
  member

openstack role assignment list \
  --project assessment-project \
  --user assessment-user \
  --names
```

Token đã được cấp thành công cho `assessment-user` trong phạm vi `assessment-project`.

## Kiểm thử Glance

Image CirrOS được kiểm tra bằng các lệnh:

```bash
openstack image show cirros
openstack image save \
  --file ~/cirros-download.qcow2 \
  cirros
md5sum ~/cirros-download.qcow2
```

Checksum của file tải về khớp với checksum lưu trong Glance:

```text
87617e24a5e30cb3b87fda8c0764838f
```

## Kiểm thử vòng đời Nova

```bash
openstack server stop test-vm
openstack server show test-vm -c status

openstack server start test-vm
openstack server show test-vm -c status

openstack server reboot test-vm
openstack server show test-vm -c status

openstack console log show test-vm
```

Instance chuyển đúng giữa `ACTIVE` và `SHUTOFF`, đồng thời trở lại `ACTIVE` sau khi reboot.

## Ghi chú troubleshooting

### ProxySQL health check

Lỗi nghiêm trọng nhất trong quá trình triển khai xuất phát từ health-check script của ProxySQL. Script gửi dữ liệu plaintext vào Unix socket sử dụng MySQL protocol, làm ProxySQL liên tục ghi lỗi và khiến Keepalived gỡ API VIP.

Hành vi ban đầu:

```bash
echo "show info" | \
  socat unix-connect:/var/lib/kolla/proxysql/admin.sock
```

Workaround trong lab:

```bash
#!/bin/bash

socat -T 2 /dev/null \
  UNIX-CONNECT:/var/lib/kolla/proxysql/admin.sock \
  >/dev/null 2>&1
```

Template đã sửa được lưu tại:

```text
templates/check_alive_proxysql.sh.j2
```

Đây chỉ là workaround cho lab và chỉ kiểm tra khả năng mở socket. Nó không phải health check hoàn chỉnh cho môi trường production.

### Vị trí log hữu ích

| Service | Thư mục log |
|---|---|
| Keystone | `/var/log/kolla/keystone/` |
| Glance | `/var/log/kolla/glance/` |
| Nova | `/var/log/kolla/nova/` |
| Placement | `/var/log/kolla/placement/` |
| Neutron | `/var/log/kolla/neutron/` |
| Cinder | `/var/log/kolla/cinder/` |
| MariaDB | `/var/log/kolla/mariadb/` |
| ProxySQL | `/var/log/kolla/proxysql/` |
| RabbitMQ | `/var/log/kolla/rabbitmq/` |
| HAProxy | `/var/log/kolla/haproxy/` |
| Horizon | `/var/log/kolla/horizon/` |
| Heat | `/var/log/kolla/heat/` |

Ví dụ xem log container:

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Status}}'
sudo docker logs --tail 200 nova_compute
sudo docker logs -f neutron_server
sudo docker logs --since 10m proxysql
sudo docker exec -it cinder_volume bash
```

## Tiêu chí hoàn thành

| Nội dung kiểm tra | Kết quả |
|---|---|
| Keystone cấp token thành công | Đạt |
| Service catalog và endpoint tồn tại | Đạt |
| Nova Compute hoạt động | Đạt |
| Hypervisor được đăng ký | Đạt |
| Neutron agent hoạt động | Đạt |
| Glance image ở trạng thái active | Đạt |
| VM ở trạng thái ACTIVE | Đạt |
| Fixed IP được cấp | Đạt |
| Floating IP truy cập được | Đạt |
| SSH vào instance thành công | Đạt |
| Cinder volume được attach | Đạt |
| Dữ liệu Cinder vẫn còn sau detach/attach | Đạt |
| Đăng nhập Horizon thành công | Đạt |
| Heat stack ở trạng thái CREATE_COMPLETE | Đạt |
| Placement allocation khớp với flavor | Đạt |

## Lưu ý bảo mật

Các file sau tuyệt đối không được commit lên public repository:

```text
/etc/kolla/passwords.yml
/etc/kolla/admin-openrc.sh
~/.ssh/openstack_lab
clouds.yaml có chứa credential
Database passwords
RabbitMQ passwords
Token và secret values
```

File `.gitignore` trong repository đã loại trừ các file secret, image, log và virtual environment thường gặp.

## Hạn chế

- Mô hình single-node, không có failover controller thực sự.
- Dùng QEMU thay vì KVM tăng tốc phần cứng.
- Cinder sử dụng local LVM thay vì distributed storage như Ceph.
- Swift, Manila, Mistral, Octavia và Barbican chưa được triển khai.
- ProxySQL health check chỉ là workaround cho lab, chưa phải SQL health check chuẩn production.
- Môi trường được thiết kế cho học tập và assessment, không dành cho production workload.

## Hướng phát triển

- Chuyển sang kiến trúc multinode.
- Thêm ba controller node để bảo đảm quorum và API high availability.
- Thêm compute, storage và network node riêng.
- Thay local LVM bằng Ceph.
- Bật TLS cho external API.
- Triển khai monitoring và centralized logging.
- Triển khai Barbican, Octavia, Manila, Swift và Mistral.
- Bổ sung script kiểm thử tự động và CI.
- Bổ sung sơ đồ kiến trúc và ảnh chụp kết quả vào repository.

## Tác giả

**QuanShin**  
Dự án phòng lab OpenStack và hạ tầng cloud cá nhân.
