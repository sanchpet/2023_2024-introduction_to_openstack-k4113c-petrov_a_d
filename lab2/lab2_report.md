# Отчёт о лабораторной работе №2

## Network, Storage, Server

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Настройка сети](#настройка-сети)
  * [Настройка хранилища](#настройка-хранилища)
  * [Создание ВМ](#создание-вм)
- [Задание](#задание)
- [Вопросы](#вопросы)
- [Заключение](#заключение)
### Описание работы
Это вторая лабораторная работа курса, в которой предлагается освоить процесс создания виртуальных машин в OpenStack и познакомиться с его сервисами. В первой части работы с помощью сервиса **Neutron** создаётся сеть провайдера и локальная сеть, в которой работают машины. Далее идет создание образа диска виртуальной машины с помощью сервиса **Glance**. На последнем этапе создаётся виртуальная машина с помощью сервиса **Nova**. Указанные действия производятся в OpenStack CLI и в веб-панели, предоставляемой сервисом **Horizon**.
### Цель работы

### Ход работы
Перед выполнением работы активируем переменные окружения, необходимые для управления Keystone и заданные в предыдущей лабораторной работе:
```bash
[root@localhost ~]# . keystonerc_admin
[root@localhost ~(keystone_admin)]#
```
#### Настройка сети
Создадим сеть провайдера с помощью команды `openstack network create`. Флаг `--external` указывает на то, что сеть будет внешней. Флаг `--default` ставит её сетью по умолчанию. `--provider-network-type flat` задает такой тип сети, в которой отсутствует внутренняя изоляция (VLAN например), т. е. все узлы сети смогут без препятствий общаться друг с другом. 

`--provider-physical-network br-ex` отвечает за реальную сеть, в которую мапируется виртуальная сеть OpenStack, в нашем случае это интерфейс `br-ex`:
```bash
[root@localhost ~(keystone_admin)]# ip a | grep -A1 'br-ex'
4: br-ex: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 26:af:50:25:d8:4c brd ff:ff:ff:ff:ff:ff
```

Выполняем команду:
```bash
[root@localhost ~(keystone_admin)]# openstack network create --external --default --provider-network-type flat --provider-physical-network br-ex sanchpet_network
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2024-03-02T11:29:34Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | f76910f0-a4e4-4e7d-a046-be64d2f5d118 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | True                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | sanchpet_network                     |
| port_security_enabled     | True                                 |
| project_id                | 2dd20895306a45ac9418cf988a170262     |
| provider:network_type     | flat                                 |
| provider:physical_network | br-ex                                |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2024-03-02T11:29:35Z                 |
+---------------------------+--------------------------------------+
```

Далее в сети провайдера создадим подсеть. Берём значение шлюза по умолчанию в системе:
```bash
IP=`/sbin/ip route | awk '/default/ { print $3 }'`
```
Затем определим адрес сети шлюза:
```bash
baseIP=`echo $IP | cut -d"." -f1-3`
```
Последний шаг перед выделением диапазона для подсети - проверить занятые IP-адреса:
```bash
[root@localhost ~(keystone_admin)]# for i in {2..254}; do echo $i; done | xargs -P 0 -I ADDR ping -c 1 -w 1 192.168.1.ADDR | grep -B1 "1 received"
--- 192.168.1.49 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.56 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.80 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.77 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.88 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.122 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.105 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.106 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
--
--- 192.168.1.148 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
```
Можно взять диапазон 110-120, например.

Подсеть создадим командой `openstack subnet create`. Параметр `--allocation-pool` отвечает за диапазон выделения адресов. `--no-dhcp` - выключение DHCP. Также указывается адрес подсети в нотации CIDR, адрес шлюза, имя сети провайдера, DNS-сервер и в конце имя подсети:
```bash
[root@localhost ~(keystone_admin)]# openstack subnet create --allocation-pool start=$baseIP.110,end=$baseIP.120 --no-dhcp --subnet-range $baseIP.0/24 --gateway $IP --network sanchpet_network --dns-nameserver 1.1.1.1 sanchpet_network_subnet
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.1.110-192.168.1.120          |
| cidr                 | 192.168.1.0/24                       |
| created_at           | 2024-03-02T11:50:05Z                 |
| description          |                                      |
| dns_nameservers      | 1.1.1.1                              |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | False                                |
| gateway_ip           | 192.168.1.1                          |
| host_routes          |                                      |
| id                   | d16a5976-8913-4d71-b7ce-287e374f48c4 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | sanchpet_network_subnet              |
| network_id           | f76910f0-a4e4-4e7d-a046-be64d2f5d118 |
| project_id           | 2dd20895306a45ac9418cf988a170262     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2024-03-02T11:50:05Z                 |
+----------------------+--------------------------------------+
```

Теперь создадим локальную сеть и подсеть в ней. Сначала просто создадим внутреннюю сеть, выдав ей имя:
```bash
[root@localhost ~(keystone_admin)]# openstack network create --internal sanchpet_private_network
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2024-03-02T11:53:02Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | b27a8e31-6131-4764-b77f-1af9927bffc8 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1442                                 |
| name                      | sanchpet_private_network             |
| port_security_enabled     | True                                 |
| project_id                | 2dd20895306a45ac9418cf988a170262     |
| provider:network_type     | geneve                               |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 93                                   |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2024-03-02T11:53:02Z                 |
+---------------------------+--------------------------------------+
```
И по аналогии с публичной подсетью создадим локальную подсеть:
```bash
[root@localhost ~(keystone_admin)]# openstack subnet create --subnet-range 172.17.22.0/24 --network sanchpet_private_network --dns-nameserver 1.1.1.1 sanchpet_private_subnet
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 172.17.22.2-172.17.22.254            |
| cidr                 | 172.17.22.0/24                       |
| created_at           | 2024-03-02T11:54:26Z                 |
| description          |                                      |
| dns_nameservers      | 1.1.1.1                              |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 172.17.22.1                          |
| host_routes          |                                      |
| id                   | 012d879e-7b45-45b9-a784-89d0d6f7fc55 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | sanchpet_private_subnet              |
| network_id           | b27a8e31-6131-4764-b77f-1af9927bffc8 |
| project_id           | 2dd20895306a45ac9418cf988a170262     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2024-03-02T11:54:26Z                 |
+----------------------+--------------------------------------+
```

Создадим маршрутизатор для объединения сетей:
```bash
[root@localhost ~(keystone_admin)]# openstack router create --external-gateway sanchpet_network sanchpet_router
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                     |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                                        |
| availability_zone_hints |                                                                                                                                                                                           |
| availability_zones      |                                                                                                                                                                                           |
| created_at              | 2024-03-02T11:59:40Z                                                                                                                                                                      |
| description             |                                                                                                                                                                                           |
| external_gateway_info   | {"network_id": "f76910f0-a4e4-4e7d-a046-be64d2f5d118", "external_fixed_ips": [{"subnet_id": "d16a5976-8913-4d71-b7ce-287e374f48c4", "ip_address": "192.168.1.117"}], "enable_snat": true} |
| flavor_id               | None                                                                                                                                                                                      |
| id                      | 31b012e2-e74f-4c7f-94c7-58bd15e38008                                                                                                                                                      |
| name                    | sanchpet_router                                                                                                                                                                           |
| project_id              | 2dd20895306a45ac9418cf988a170262                                                                                                                                                          |
| revision_number         | 3                                                                                                                                                                                         |
| routes                  |                                                                                                                                                                                           |
| status                  | ACTIVE                                                                                                                                                                                    |
| tags                    |                                                                                                                                                                                           |
| updated_at              | 2024-03-02T11:59:40Z                                                                                                                                                                      |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
Обратим внимание, что роутер получил адрес из диапазона, который мы указывали выдачи для подсети провайдера -  192.168.1.117.
Добавим роутеру созданную локальную подсеть:
```bash
[root@localhost ~(keystone_admin)]# openstack router add subnet sanchpet_router sanchpet_private_subnet
```
На этом настройка сети завершена, перейдём к созданию машин.
#### Настройка хранилища
Создадим флейвор ВМ. Флейвор - шаблон, который задает "железные" настройки (CPU, RAM, хранилище) для ВМ. Зададим размер RAM в 256 Мб:
```bash
[root@localhost ~(keystone_admin)]# openstack flavor create --ram=256 sanchpet_tiny_flavor
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| description                | None                                 |
| disk                       | 0                                    |
| id                         | 56765887-f764-4fa7-8b2a-e065680fe34c |
| name                       | sanchpet_tiny_flavor                 |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 256                                  |
| rxtx_factor                | 1.0                                  |
| swap                       |                                      |
| vcpus                      | 1                                    |
+----------------------------+--------------------------------------+
```
Сгенерируем ключ для доступа к машине:
```bash
[root@localhost ~(keystone_admin)]# openstack keypair create --public-key ~/.ssh/id_rsa.pub my_key
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| created_at  | None                                            |
| fingerprint | 0e:30:aa:0a:b4:1b:d9:f2:14:96:f0:c2:42:14:3b:9d |
| id          | my_key                                          |
| is_deleted  | None                                            |
| name        | my_key                                          |
| type        | ssh                                             |
| user_id     | a5b8b6dd7ffa47eabfda49c8203ecafe                |
+-------------+-------------------------------------------------+
```
Качаем образ Cirros в OpenStack Glance:
```bash
[root@localhost ~(keystone_admin)]# yum install wget -y
[root@localhost ~(keystone_admin)]# wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
```
И создадим из него образ диска в формате qcow2:
```bash
[root@localhost ~(keystone_admin)]# openstack image create --disk-format qcow2 --public --file ./cirros-0.6.2-x86_64-disk.img sanchpet_cirros_image
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                     |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------+
| container_format | bare                                                                                                                                                      |
| created_at       | 2024-03-02T12:20:40Z                                                                                                                                      |
| disk_format      | qcow2                                                                                                                                                     |
| file             | /v2/images/a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e/file                                                                                                      |
| id               | a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e                                                                                                                      |
| min_disk         | 0                                                                                                                                                         |
| min_ram          | 0                                                                                                                                                         |
| name             | sanchpet_cirros_image                                                                                                                                     |
| owner            | 2dd20895306a45ac9418cf988a170262                                                                                                                          |
| properties       | os_hidden='False', owner_specified.openstack.md5='', owner_specified.openstack.object='images/sanchpet_cirros_image', owner_specified.openstack.sha256='' |
| protected        | False                                                                                                                                                     |
| schema           | /v2/schemas/image                                                                                                                                         |
| status           | queued                                                                                                                                                    |
| tags             |                                                                                                                                                           |
| updated_at       | 2024-03-02T12:20:40Z                                                                                                                                      |
| visibility       | public                                                                                                                                                    |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------+
```
Теперь из образа можно создать блочное устройство в Cinder командой `openstack volume create`. `--size` отвечает за размер устройства в GB, параметр `--bootable` указывает на то, что с устройства возможна загрузка (т. к. мы разметили на нём образ ОС)
```bash
[root@localhost ~(keystone_admin)]# openstack volume create --size 2 --image sanchpet_cirros_image --bootable sanchpet_disk
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-03-02T12:21:33.904713           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 7209fbab-6d0c-4a25-b8b5-3550f7789506 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | sanchpet_disk                        |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 2                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | iscsi                                |
| updated_at          | None                                 |
| user_id             | a5b8b6dd7ffa47eabfda49c8203ecafe     |
+---------------------+--------------------------------------+
```
#### Создание ВМ
Перейдём в панель Horizon и создадим ВМ. На этапе выбора источника укажем наш диск:

![img1](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302152730.png)

В качестве флейвора укажем созданный нами:

![img2](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302153037.png)

И в качестве сети укажем созданную нами приватную сеть:

![img3](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302153116.png)

Надо заметить, что в разделе "Key Pair" в настройки машины уже помещён созданный публичный ключ. Жмём "Launch Instance" и наблюдаем в интерфейсе процесс создания машины. После создания мы можем открыть VNC консоль и проверить, какой адрес получила машина:

![img4](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302153906.png)

Видим, что IP-адрес получен.
### Задание
В Horizon создадим ещё одну пару из приватной сети и подсети. В разделе "Network" -> "Networks" создадим сеть, на этом же экране прожмём галочку "Create subnet" и введём параметры подсети:

![img5](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302165537.png)

Подключаем подсеть к маршрутизатору и проверяем топологию сети:

![img6](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302170620.png)

Следующим образом создадим копию созданного ранее блочного устройства:
```bash
[root@localhost ~(keystone_admin)]# openstack volume list
+--------------------------------------+---------------+--------+------+----------------------------------------+
| ID                                   | Name          | Status | Size | Attached to                            |
+--------------------------------------+---------------+--------+------+----------------------------------------+
| 7209fbab-6d0c-4a25-b8b5-3550f7789506 | sanchpet_disk | in-use |    2 | Attached to sanchpet_vm_1 on /dev/vda  |
+--------------------------------------+---------------+--------+------+----------------------------------------+
[root@localhost ~(keystone_admin)]# openstack volume snapshot create --volume 7209fbab-6d0c-4a25-b8b5-3550f7789506 sanchpet_disk_snap_1 --force
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| created_at  | 2024-03-02T14:14:14.263485           |
| description | None                                 |
| id          | d9282d11-d507-4854-a457-9b0d32b0bee3 |
| name        | sanchpet_disk_snap_1                 |
| properties  |                                      |
| size        | 2                                    |
| status      | creating                             |
| updated_at  | None                                 |
| volume_id   | 7209fbab-6d0c-4a25-b8b5-3550f7789506 |
+-------------+--------------------------------------+
[root@localhost ~(keystone_admin)]# openstack volume snapshot list
+--------------------------------------+----------------------+-------------+-----------+------+
| ID                                   | Name                 | Description | Status    | Size |
+--------------------------------------+----------------------+-------------+-----------+------+
| d9282d11-d507-4854-a457-9b0d32b0bee3 | sanchpet_disk_snap_1 | None        | available |    2 |
+--------------------------------------+----------------------+-------------+-----------+------+
[root@localhost ~(keystone_admin)]# openstack volume create --snapshot d9282d11-d507-4854-a457-9b0d32b0bee3 sanchpet_disk_2
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | true                                 |
| consistencygroup_id | None                                 |
| created_at          | 2024-03-02T14:15:00.286000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 1dcfc260-5be5-4ddf-a59a-c5b512eb9d77 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | sanchpet_disk_2                      |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 2                                    |
| snapshot_id         | d9282d11-d507-4854-a457-9b0d32b0bee3 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | iscsi                                |
| updated_at          | None                                 |
| user_id             | a5b8b6dd7ffa47eabfda49c8203ecafe     |
+---------------------+--------------------------------------+
[root@localhost ~(keystone_admin)]# openstack volume list
+--------------------------------------+-----------------+-----------+------+----------------------------------------+
| ID                                   | Name            | Status    | Size | Attached to                            |
+--------------------------------------+-----------------+-----------+------+----------------------------------------+
| 1dcfc260-5be5-4ddf-a59a-c5b512eb9d77 | sanchpet_disk_2 | available |    2 |                                        |
| 7209fbab-6d0c-4a25-b8b5-3550f7789506 | sanchpet_disk   | in-use    |    2 | Attached to sanchpet_vm_1 on /dev/vda  |
+--------------------------------------+-----------------+-----------+------+----------------------------------------+
```
Через CLI создадим новую ВМ с нашим новым диском, старым флейвором и созданной приватной сетью:
```bash
[root@localhost ~(keystone_admin)]# openstack server create --volume sanchpet_disk_2 --flavor sanchpet_tiny_flavor --network sanchpet_private_network_2 sanchpet_vm_2
+-------------------------------------+-------------------------------------------------------------+
| Field                               | Value                                                       |
+-------------------------------------+-------------------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                                      |
| OS-EXT-AZ:availability_zone         |                                                             |
| OS-EXT-SRV-ATTR:host                | None                                                        |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                                        |
| OS-EXT-SRV-ATTR:instance_name       |                                                             |
| OS-EXT-STS:power_state              | NOSTATE                                                     |
| OS-EXT-STS:task_state               | scheduling                                                  |
| OS-EXT-STS:vm_state                 | building                                                    |
| OS-SRV-USG:launched_at              | None                                                        |
| OS-SRV-USG:terminated_at            | None                                                        |
| accessIPv4                          |                                                             |
| accessIPv6                          |                                                             |
| addresses                           |                                                             |
| adminPass                           | x8ZntxR6kUcw                                                |
| config_drive                        |                                                             |
| created                             | 2024-03-02T14:18:40Z                                        |
| flavor                              | sanchpet_tiny_flavor (56765887-f764-4fa7-8b2a-e065680fe34c) |
| hostId                              |                                                             |
| id                                  | 255c76ac-169d-4f80-9bf2-66ef3f387f9c                        |
| image                               | N/A (booted from volume)                                    |
| key_name                            | None                                                        |
| name                                | sanchpet_vm_2                                               |
| progress                            | 0                                                           |
| project_id                          | 2dd20895306a45ac9418cf988a170262                            |
| properties                          |                                                             |
| security_groups                     | name='default'                                              |
| status                              | BUILD                                                       |
| updated                             | 2024-03-02T14:18:40Z                                        |
| user_id                             | a5b8b6dd7ffa47eabfda49c8203ecafe                            |
| volumes_attached                    |                                                             |
+-------------------------------------+-------------------------------------------------------------+
```
Внутри Horizon проверим наличие ВМ, проверим ее адрес в новой приватной сети через VNC:

![img7](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302172103.png)


![img8](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab2/img/Pasted%20image%2020240302172034.png)

### Дополнительные вопросы
1. *Что именно сервис с помощью Keystone проверяет в токене пользователя, когда тот пытается осуществить операцию по отношению к этому сервису?*
   **Сервис проверяет владельца токена, роли и разрешения токена, чтобы определить, может ли пользователь выполнять запрашиваемую операцию. Также проверяется срок действия токена, т. к. токен обычно имеет ограниченный срок действия и может истечь. Также в различных форматах токена могут содержаться подписи.**
2. *При создании ВМ, Nova первым делом идет в Keystone, проверяет токен и т.д. Как думаете, к эндпоинту какого сервиса Nova идет следом?*
   **После того, как Keystone провел аутентификацию пользователя и выдал токен, Nova обращается к Glance, чтобы получить образ виртуальной машины, который указал пользователь при создании.** 
### Заключение
В ходе работы создана сеть провайдера и локальная сеть, создан образ диска и создана виртуальная машина. Получены навыки управления виртуальными машинами в OpenStack CLI и в веб-интерфейсе Horizon. Полученные навыки в дальнейшем могут быть использованы для управления виртуальными машинами на базе OpenStack. 

