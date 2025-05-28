# Triển khai Neutron trên Controller và Neutron Node

## Prerequisites

### 1. Chuẩn bị Cơ sở dữ liệu cho Neutron trên node controller
```shell
# Đăng nhập vào MySQL
mysql
```
```sql
-- Tạo database neutron
CREATE DATABASE neutron;

-- Cấp quyền cho user neutron
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'OpenStack';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'OpenStack';

-- Thoát MySQL
EXIT;
```

### 2. Chuẩn bị Keystone trên node controller
```shell
# Load biến môi trường admin
. admin-openrc
```
```shell
# Tạo user neutron và các endpoint
openstack user create --domain default --password 'OpenStack' neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public https://neutron.kbuor.io.vn:9696
openstack endpoint create --region RegionOne network internal https://neutron.kbuor.io.vn:9696
openstack endpoint create --region RegionOne network admin https://neutron.kbuor.io.vn:9696
```

---

## Cài đặt và cấu hình Neutron trên node Neutron Node

### 3. Cài đặt package Neutron
```shell
apt install -y neutron-server neutron-plugin-ml2 neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

### 4. Cấu hình các file cấu hình Neutron
#### neutron.conf
```shell
cat <<'EOF' > /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:OpenStack@controller01.openstack.local
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[database]
connection = mysql+pymysql://neutron:OpenStack@controller01.openstack.local/neutron
[keystone_authtoken]
www_authenticate_uri = https://keystone.kbuor.io.vn:5000
auth_url = https://keystone.kbuor.io.vn:5000
memcached_servers = controller01.openstack.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = OpenStack
[nova]
auth_url = https://keystone.kbuor.io.vn:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = OpenStack
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
EOF
```

#### ml2_conf.ini
```shell
cat <<'EOF' > /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
EOF
```
## add card mạng lần luotj theo đúng thứ tự, provider >> netplan apply >> self service >> netplan apply

#### Cấu hình Open vSwitch
```shell
ovs-vsctl add-br provider-br-01
ovs-vsctl add-port provider-br-01 ens224
```
### add bridge vào file openstack.yaml

```
network:
  version: 2
  renderer: networkd
  bridges:
    provider-br-01:
      dhcp4: no
      interfaces:
        - ens224
  ethernets:
    ens256:
      dhcp4: no
      dhcp6: no
      addresses:
        - 12.12.12.31/24
    ens224:
      dhcp4: no
      dhcp6: no
      link-local: []
    ens192:
      dhcp4: no
      dhcp6: no
      addresses:
        - 10.20.30.31/24
      routes:
        - to: default
          via: 10.20.30.1
      nameservers:
        search:
          - openstack.local
        addresses:
          - 10.20.30.250
```
#### openvswitch_agent.ini
```shell
cat <<'EOF' > /etc/neutron/plugins/ml2/openvswitch_agent.ini
[agent]
tunnel_types = vxlan
l2_population = true
[ovs]
bridge_mappings = provider-network-01:provider-br-01
local_ip = 13.28.4.61
[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
EOF
```

#### l3_agent.ini
```shell
cat <<'EOF' > /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = openvswitch
EOF
```

#### dhcp_agent.ini
```shell
cat <<'EOF' > /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
EOF
```

#### metadata_agent.ini
```shell
cat <<'EOF' > /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = nova.tpcoms.ai
metadata_proxy_shared_secret = OpenStack
EOF
```

---

## Cập nhật Nova để tích hợp Neutron (trên Controller Node)

### 5. Chỉnh sửa file `/etc/nova/nova.conf` thêm vào cuối file
```ini
[neutron]
auth_url = https://keystone.kbuor.io.vn:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = OpenStack
service_metadata_proxy = true
metadata_proxy_shared_secret = OpenStack
```

---

## Đồng bộ database và khởi động dịch vụ Neutron

### 6. Đồng bộ CSDL trên node Neutron
```shell
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

### 7. Khởi động các dịch vụ trên node Neutron
```shell
systemctl enable neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl restart neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
```

---

## Chạy Neutron API bằng Apache HTTPD trên node neutron

### 8. Vô hiệu hóa neutron-server và tạo WSGI script
```shell
systemctl disable neutron-server
systemctl stop neutron-server

apt install apache2 libapache2-mod-wsgi-py3 -y
```

### 9. Tạo VirtualHost chạy Neutron API qua HTTPS
```shell
cat <<'EOF' > /etc/apache2/sites-available/neutron-https.conf
<VirtualHost *:9696>
    ServerName neutron.kbuor.io.vn

    SSLEngine on
    SSLCertificateFile /etc/ssl/openstack/openstack.crt
    SSLCertificateKeyFile /etc/ssl/openstack/openstack.key

    WSGIDaemonProcess neutron user=neutron group=neutron processes=4 threads=10 display-name=%{GROUP}
    WSGIProcessGroup neutron
    WSGIScriptAlias / /usr/bin/neutron-api

    <Directory /usr/bin>
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/neutron_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/neutron_ssl_access.log combined
</VirtualHost>
EOF

echo 'Listen 9696' >> /etc/apache2/ports.conf
```

### 10. Bật HTTPS cho Neutron và khởi động Apache
```shell
a2enmod ssl
a2ensite neutron-https
systemctl restart apache2
systemctl reload apache2
```
đoạn này check trên controller là ra XXX , chưa cười

### trên node neutron

```
vi /etc/neutron/neutron.conf
```
thêm nội dung sau vaof đầu file
```
[DEFAULT]
bind_host = 127.0.0.1
bind_port = 9697
```

```
[DEFAULT]
bind_host = 127.0.0.1
bind_port = 9697
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:OpenStack@controller01.openstack.local
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[database]
connection = mysql+pymysql://neutron:OpenStack@controller01.openstack.local/neutron
[keystone_authtoken]
www_authenticate_uri = https://keystone.tpcoms.ai:5000
auth_url = https://keystone.tpcoms.ai:5000
memcached_servers = controller01.openstack.local:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = OpenStack
[nova]
auth_url = https://keystone.tpcoms.ai:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = OpenStack
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

```

```
systemctl restart neutron-server
systemctl enable  neutron-server
```

quay lại controller check là thấy nó cười, ddoanj nayf chuwa cos compute01 , 02

```
root@controller01:~# openstack network agent list
+----------------------+--------------------+----------------------+-------------------+-------+-------+-----------------------+
| ID                   | Agent Type         | Host                 | Availability Zone | Alive | State | Binary                |
+----------------------+--------------------+----------------------+-------------------+-------+-------+-----------------------+
| 3d3f7fd9-7155-49e8-  | DHCP agent         | neutron01.openstack. | nova              | :-)   | UP    | neutron-dhcp-agent    |
| 8d77-712956dc3c22    |                    | local                |                   |       |       |                       |
| 854dc5fd-fde0-4f03-  | Open vSwitch agent | compute01.openstack. | None              | :-)   | UP    | neutron-openvswitch-  |
| bbdd-808b67f9ed59    |                    | local                |                   |       |       | agent                 |
| 95dd44cc-e7f5-4b72-  | L3 agent           | neutron01.openstack. | nova              | :-)   | UP    | neutron-l3-agent      |
| 9995-e612a4e682fc    |                    | local                |                   |       |       |                       |
| bcb573ad-4444-4b53-  | Open vSwitch agent | compute02.openstack. | None              | :-)   | UP    | neutron-openvswitch-  |
| baee-f1a4f907a8b5    |                    | local                |                   |       |       | agent                 |
| cdf0a76c-6660-4824-  | Open vSwitch agent | neutron01.openstack. | None              | :-)   | UP    | neutron-openvswitch-  |
| a58d-5d7d46bf25e3    |                    | local                |                   |       |       | agent                 |
| f07052a3-1f47-49e0-  | Metadata agent     | neutron01.openstack. | None              | :-)   | UP    | neutron-metadata-     |
| 835b-eb703f323a55    |                    | local                |                   |       |       | agent                 |
+----------------------+--------------------+----------------------+-------------------+-------+-------+-----------------------+
root@controller01:~#

```

### XOng neutron, giowf qua cài compute01, 02

## trên  compute01

cài 
```
apt install -y neutron-openvswitch-agent
```

```
vi /etc/neutron/neutron.conf

```

```
[DEFAULT]
transport_url = rabbit://openstack:OpenStack@controller01.openstack.local
core_plugin = ml2
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cache]
[cors]
[database]
[healthcheck]
[ironic]
[keystone_authtoken]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[oslo_reports]
[placement]
[privsep]
[profiler]
[quotas]
[ssl]
```

thêm card mạng lần lượt provider >> self service

#### Cấu hình Open vSwitch
```shell
ovs-vsctl add-br provider-br-01
ovs-vsctl add-port provider-br-01 ens224
```
### add bridge vào file openstack.yaml

```
network:
  version: 2
  renderer: networkd
  bridges:
    provider-br-01:
      dhcp4: no
      interfaces:
        - ens224
  ethernets:
    ens256:
      dhcp4: no
      dhcp6: no
      addresses:
        - 12.12.12.31/24
    ens224:
      dhcp4: no
      dhcp6: no
      link-local: []
    ens192:
      dhcp4: no
      dhcp6: no
      addresses:
        - 10.20.30.31/24
      routes:
        - to: default
          via: 10.20.30.1
      nameservers:
        search:
          - openstack.local
        addresses:
          - 10.20.30.250
```

#### openvswitch_agent.ini

```shell
cat <<'EOF' > /etc/neutron/plugins/ml2/openvswitch_agent.ini
[agent]
tunnel_types = vxlan
l2_population = true
[ovs]
bridge_mappings = provider-network-01:provider-br-01
local_ip = 12.12.12.21
[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
EOF
```

```
systemctl restart neutron-openvswitch-agent.service
systemctl enable neutron-openvswitch-agent.service
```

quay qua controller check lại 

```
openstack network agent list
```




---

## Ghi chú
- Đảm bảo các hostname như `controller01.openstack.local`, `neutron.kbuor.io.vn`, và `keystone.kbuor.io.vn` đã được khai báo đúng trong DNS hoặc `/etc/hosts`.
- Địa chỉ `ens224`, `13.28.4.61` cần thay đổi phù hợp với hạ tầng mạng thực tế của bạn.
