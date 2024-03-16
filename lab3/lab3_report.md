# Отчёт о лабораторной работе №2

## Работа с Openstack API

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Запрос токена в Keystone](#запрос-токена-в-keystone)
  * [Проверка валидности токена](проверка-валидности-токена)
  * [Создание ВМ через Nova API](#создание-вм-через-nova-api)
- [Задание](#задание)
- [Вопросы](#вопросы)
- [Заключение](#заключение)
### Описание работы
Это третья лабораторная работа курса, в рамках которой предлагается ознакомиться с OpenStack API. В этой работе будет получен токен, с помощью которого можно простым способом аутентифицировать запросы к API и совершать действия без участия CLI или веб-панели. 
### Цель работы
Получить практический опыт работы с OpenStack API.
### Ход работы
Запустим виртуальную машину OpenStack и убедимся, что сетевой интерфейс по адресу `192.168.1.56` доступен для работы.
#### Запрос токена в Keystone
Чтобы получить токен, нам потребуется отправить запрос на авторизационный эндпоинт, в котором будет передаваться json-файл с данными для авторизации. 
Воспользуемся возможностями утилиты `curl` и напишем простой скрипт, который будет получать токен и экспортировать его в глобальную переменную `OS_TOKEN`, чтобы мы могли сразу использовать её в других запросах из наших скриптов или из командной строки:
```bash
#!/usr/bin/env bash

[[ -z ${OS_USERNAME} ]] && echo "Export keystone admin variables to use this script" && exit 1

cat << EOF > /tmp/auth.json
{
    "auth": {
        "identity": {
            "methods": ["password"],
            "password": {
                "user": {
                    "domain": {
                        "id": "${OS_USER_DOMAIN_NAME,}"
                    },
                    "name": "${OS_USERNAME}",
                    "password": "${OS_PASSWORD}"
                }
            }
        },
        "scope": {
            "project": {
                "name": "${OS_PROJECT_NAME}",
                "domain": {
                    "id": "${OS_PROJECT_DOMAIN_NAME,}"
                }
            }
        }
    }
}
EOF

export OS_TOKEN=$(curl -si -d @/tmp/auth.json -H "Content-type: application/json" $OS_AUTH_URL/auth/tokens | awk '/X-Subject-Token/ {print $2}')
rm -f /tmp/auth.json

echo "Your token is: ${OS_TOKEN}"
echo "Now you can use global variable \${OS_TOKEN} for authentication."
```
Это максимально простой скрипт, который генерирует json-файл для авторизации на основе глобальных переменных keystone admin, отправляет запрос на API, экспортируя переменную токена. Файл для json-авторизации не сохраняется, и я могу быть уверен в том, что пароль административного пользователя хранится только в одном файле на машине. 
Выполним скрипт (обращу внимание на то, что мы производим операцию `source` с помощью символа `.` перед скриптом, чтобы передать значение переменной в нашу оболочку):
```bash
[root@localhost lab3(keystone_admin)]# . ./get_local_openstack_token.sh
Your token is: gAAAAABl9XBpPzmr2B7a1LPeEVJ5zd_ZJCrwdXB66PbTbvXLYgQmzA8BueO_wIZPSCVnM__FI8-2ytHvM3Qq5l4yevnlt5_CmKVbE7i1F2SBnR-VTiidbFxYyMd2FUnRT0qh6M8u1y-81376tvBKI0M5hUkUMQt_G7drLD567ZmAvzBC_aA5X0Q
Now you can use global variable ${OS_TOKEN} for authentication.
```
Проверим, что токен у нас есть:
```bash
[root@localhost lab3(keystone_admin)]# echo ${OS_TOKEN}
gAAAAABl9XBpPzmr2B7a1LPeEVJ5zd_ZJCrwdXB66PbTbvXLYgQmzA8BueO_wIZPSCVnM__FI8-2ytHvM3Qq5l4yevnlt5_CmKVbE7i1F2SBnR-VTiidbFxYyMd2FUnRT0qh6M8u1y-81376tvBKI0M5hUkUMQt_G7drLD567ZmAvzBC_aA5X0Q
```
#### Проверка валидности токена
Проверить валидность токена можно, отправив любой запрос к одному из сервисов Openstack.
Так можно получить список доменов:
```bash
[root@localhost lab3(keystone_admin)]# curl -s  -H "X-Auth-Token:${OS_TOKEN}" "${OS_AUTH_URL}/domains" | jq
{
  "domains": [
    {
      "id": "default",
      "name": "Default",
      "description": "The default domain",
      "enabled": true,
      "tags": [],
      "options": {},
      "links": {
        "self": "http://localhost:5000/v3/domains/default"
      }
    }
  ],
  "links": {
    "next": null,
    "self": "http://localhost:5000/v3/domains",
    "previous": null
  }
}
```
Получить список образов Nova:
```bash
[root@localhost lab3(keystone_admin)]# curl -s  -H "X-Auth-Token:${OS_TOKEN}" "http://192.168.1.56:8774/v2.1/images" | jq
{
  "images": [
    {
      "id": "a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e",
      "name": "sanchpet_cirros_image",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8774/v2.1/images/a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8774/images/a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e"
        },
        {
          "rel": "alternate",
          "type": "application/vnd.openstack.image",
          "href": "http://192.168.1.56:9292/images/a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e"
        }
      ]
    }
  ]
}
```
Ну и проверим список блочных устройств в Cinder:
```bash
[root@localhost lab3(keystone_admin)]# curl -s  -H "X-Auth-Token:${OS_TOKEN}" "http://192.168.1.56:8776/v3/volumes" | jq
{
  "volumes": [
    {
      "id": "1dcfc260-5be5-4ddf-a59a-c5b512eb9d77",
      "name": "sanchpet_disk_2",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8776/v3/volumes/1dcfc260-5be5-4ddf-a59a-c5b512eb9d77"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8776/volumes/1dcfc260-5be5-4ddf-a59a-c5b512eb9d77"
        }
      ]
    },
    {
      "id": "7209fbab-6d0c-4a25-b8b5-3550f7789506",
      "name": "sanchpet_disk",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8776/v3/volumes/7209fbab-6d0c-4a25-b8b5-3550f7789506"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8776/volumes/7209fbab-6d0c-4a25-b8b5-3550f7789506"
        }
      ]
    }
  ]
}

```
#### Создание ВМ через Nova API
Применим такой простой скрипт для создания ВМ:
```bash
#!/usr/bin/env bash

AUTH_TOKEN="${1-}"
IMAGE_ID="${2-}"
FLAVOR_ID="${3-}"
NETWORK_ID="${4-}"

curl -i -X POST "http://192.168.1.56:8774/v2.1/servers" \
     -H "X-Auth-Token: $AUTH_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "server": {
             "name": "my_new_instance",
             "imageRef": "'$IMAGE_ID'",
             "flavorRef": "'$FLAVOR_ID'",
             "networks": [
               {
                 "uuid": "'$NETWORK_ID'"
               }
             ]
           }
         }'
```
Выполним его, взяв параметры соответствующих ID из Horizon:
```bash
./create_nova_vm.sh "${OS_TOKEN}" "a54f07fc-f30c-4fd5-bfdb-9af3b0ebd64e" "56765887-f764-4fa7-8b2a-e065680fe34c" "b27a8e31-6131-4764-b77f-1af9927bffc8"
HTTP/1.1 202 Accepted
Date: Sat, 16 Mar 2024 11:05:56 GMT
Server: Apache
Content-Length: 376
location: http://192.168.1.56:8774/v2.1/servers/6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5
OpenStack-API-Version: compute 2.1
X-OpenStack-Nova-API-Version: 2.1
Vary: OpenStack-API-Version,X-OpenStack-Nova-API-Version
x-openstack-request-id: req-c4bc3213-5eec-4c88-9891-940c79503c44
x-compute-request-id: req-c4bc3213-5eec-4c88-9891-940c79503c44
Content-Type: application/json
```
Прогоним тело ответа через jq для красивого вывода:
```bash
{
  "server": {
    "id": "6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5",
    "links": [
      {
        "rel": "self",
        "href": "http://192.168.1.56:8774/v2.1/servers/6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5"
      },
      {
        "rel": "bookmark",
        "href": "http://192.168.1.56:8774/servers/6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5"
      }
    ],
    "OS-DCF:diskConfig": "MANUAL",
    "security_groups": [
      {
        "name": "default"
      }
    ],
    "adminPass": "TpmNCqwd6GHC"
  }
}
```
Убедимся, что сервер появился:
```bash
[root@localhost lab3(keystone_admin)]# curl -s  -H "X-Auth-Token:${OS_TOKEN}" "http://192.168.1.56:8774/v2.1/servers" | jq
{
  "servers": [
    {
      "id": "6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5",
      "name": "my_new_instance",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8774/v2.1/servers/6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8774/servers/6f7d53df-2e1b-4df5-9a28-c5ca4105d1b5"
        }
      ]
    },
    {
      "id": "255c76ac-169d-4f80-9bf2-66ef3f387f9c",
      "name": "sanchpet_vm_2",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8774/v2.1/servers/255c76ac-169d-4f80-9bf2-66ef3f387f9c"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8774/servers/255c76ac-169d-4f80-9bf2-66ef3f387f9c"
        }
      ]
    },
    {
      "id": "a93d3e52-be70-4ffc-a0e5-f1013285fafd",
      "name": "sanchpet_vm_1",
      "links": [
        {
          "rel": "self",
          "href": "http://192.168.1.56:8774/v2.1/servers/a93d3e52-be70-4ffc-a0e5-f1013285fafd"
        },
        {
          "rel": "bookmark",
          "href": "http://192.168.1.56:8774/servers/a93d3e52-be70-4ffc-a0e5-f1013285fafd"
        }
      ]
    }
  ]
}
```
### Задание
Попробуем настроить наш собственный single-node CEPH и подключить его в качестве бэкенда для Cinder, чтобы хранить там блочные устройства для машин Nova.

Создадим виртуальную машину в Virtualbox с ОС Alma Linux 9.3 и произведём настройку single-node CEPH:
```bash
[root@localhost ~]# dnf update -y

[root@localhost ~]# yum install ceph-common

[root@localhost ~]# dnf install -y centos-release-ceph-pacific

[root@localhost ~]# curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm

[root@localhost ~]# chmod +x cephadm

[root@localhost ~]# ./cephadm install
```
Отдельно приведу вывод процесса применения bootstrap-решения с настройками для single-host (указан IP машины, полученный через bridge). Описываются компоненты, которые запускаются вместе с CEPH:
```bash
[root@localhost ~]# cephadm bootstrap --single-host-defaults --mon-ip 192.168.1.114
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chronyd.service is enabled and running
Repeating the final host check...
podman (/usr/bin/podman) version 4.6.1 is present
systemctl is present
lvcreate is present
Unit chronyd.service is enabled and running
Host looks OK
Cluster fsid: cb66b89c-e38b-11ee-9c57-0800272deeb9
Verifying IP 192.168.1.114 port 3300 ...
Verifying IP 192.168.1.114 port 6789 ...
Mon IP `192.168.1.114` is in CIDR network `192.168.1.0/24`
Mon IP `192.168.1.114` is in CIDR network `192.168.1.0/24`
Internal network (--cluster-network) has not been provided, OSD replication will default to the public_network
Adjusting default settings to suit single-host cluster...
Pulling container image quay.io/ceph/ceph:v16...
Ceph version: ceph version 16.2.15 (618f440892089921c3e944a991122ddc44e60516) pacific (stable)
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
firewalld ready
Enabling firewalld service ceph-mon in current zone...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting public_network to 192.168.1.0/24 in mon config section
Wrote config to /etc/ceph/ceph.conf
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Creating mgr...
Verifying port 9283 ...
firewalld ready
Enabling firewalld service ceph in current zone...
firewalld ready
Enabling firewalld port 9283/tcp in current zone...
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/15)...
mgr not available, waiting (2/15)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for mgr epoch 5...
mgr epoch 5 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to /etc/ceph/ceph.pub
Adding key to root@localhost authorized_keys...
Adding host localhost...
Deploying mon service with default placement...
Deploying mgr service with default placement...
Deploying crash service with default placement...
Deploying prometheus service with default placement...
Deploying grafana service with default placement...
Deploying node-exporter service with default placement...
Deploying alertmanager service with default placement...
Enabling the dashboard module...
Waiting for the mgr to restart...
Waiting for mgr epoch 9...
mgr epoch 9 is available
Generating a dashboard self-signed certificate...
Creating initial admin user...
Fetching dashboard port number...
firewalld ready
Enabling firewalld port 8443/tcp in current zone...
Ceph Dashboard is now available at:

             URL: https://localhost.localdomain:8443/
            User: admin
        Password: l3fvhuso83

Enabling client.admin keyring and conf on hosts with "admin" label
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

        sudo /usr/sbin/cephadm shell --fsid cb66b89c-e38b-11ee-9c57-0800272deeb9 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

        sudo /usr/sbin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

        ceph telemetry on

For more information see:

        https://docs.ceph.com/en/pacific/mgr/telemetry/

Bootstrap complete.
```
В этом выводе есть полезная информация, например, данные для входа в Ceph Dashboard по адресу https://192.168.1.114:8443

Проверим установку:
```bash
[root@localhost ~]# ceph -s
  cluster:
    id:     cb66b89c-e38b-11ee-9c57-0800272deeb9
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 2

  services:
    mon: 1 daemons, quorum localhost (age 2m)
    mgr: localhost.eqhtiv(active, since 37s), standbys: localhost.hizucc
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```
Как видно, CEPH пока не в чем хранить данные. Добавим к виртуальной машине новый диск через VirtualBox:
```bash
[root@localhost ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0 30.2G  0 disk
├─sda1               8:1    0    1G  0 part /boot
└─sda2               8:2    0 29.2G  0 part
  ├─almalinux-root 253:0    0 26.1G  0 lvm  /var/lib/containers/storage/overlay
  │                                         /
  └─almalinux-swap 253:1    0    3G  0 lvm  [SWAP]
sdb                  8:16   0   20G  0 disk
sr0                 11:0    1 1024M  0 rom
```
Создадим пул и добавим диск:
```bash
[root@localhost ~]# ceph osd pool create mypool 1
pool 'mypool' created
[root@localhost ~]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...
[root@localhost ~]# sudo ceph orch daemon add osd localhost:/dev/sdb
Created osd(s) 0 on host 'localhost'
[root@localhost ~]# ceph -s
  cluster:
    id:     cb66b89c-e38b-11ee-9c57-0800272deeb9
    health: HEALTH_WARN
            Reduced data availability: 1 pg inactive, 1 pg peering
            OSD count 1 < osd_pool_default_size 2

  services:
    mon: 1 daemons, quorum localhost (age 4m)
    mgr: localhost.eqhtiv(active, since 3m), standbys: localhost.hizucc
    osd: 1 osds: 1 up (since 3s), 1 in (since 17s)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   290 MiB used, 20 GiB / 20 GiB avail
    pgs:     100.000% pgs not active
             1 creating+peering
```
Теперь есть место, где хранить данные. Перейдем к интеграции CEPH в Openstack. Отправим на машину Openstack настройки для подключения к CEPH:
```bash
[root@localhost ~]# ssh root@192.168.1.56 sudo tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
# minimal ceph.conf for cb66b89c-e38b-11ee-9c57-0800272deeb9
[global]
        fsid = cb66b89c-e38b-11ee-9c57-0800272deeb9
        mon_host = [v2:192.168.1.114:3300/0,v1:192.168.1.114:6789/0]
```
Создадим пользователя для cinder:
```bash
[root@localhost ~]# ceph auth get-or-create client.cinder
[client.cinder]
        key = AQDujPVllrykBhAAWUnqb6SfxrEIRb1F8GxvXA==
```
На машине с Openstack Обновим конфигурацию для Cinder:
```bash
[root@localhost ~(keystone_admin)]# head -n15 /etc/cinder/cinder.conf
[DEFAULT]
enabled_backends = ceph
glance_api_version = 2

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = mypool
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = 7138ce00-c5bc-4ab1-b0f4-833ce1948ac9
```
Перезапустим Cinder:
```bash
[root@localhost ~(keystone_admin)]# systemctl restart openstack-cinder-volume.service
```
И попробуем создать новое блочное устройство:
```bash
root@localhost ~(keystone_admin)]# openstack volume create --size 2 --image sanchpet_cirros_image --bootable sanchpet_disk3
```
Однако из-за ошибок в конфигурации была получена ошибка:
```bash
[root@localhost ~(keystone_admin)]# openstack volume list
+--------------------------------------+-----------------+--------+------+----------------------------------------+
| ID                                   | Name            | Status | Size | Attached to                            |
+--------------------------------------+-----------------+--------+------+----------------------------------------+
| f2b94df7-52ce-4066-aff7-d0d8f60afdd8 | sanchpet_disk3  | error  |    2 |                                        |
| 1dcfc260-5be5-4ddf-a59a-c5b512eb9d77 | sanchpet_disk_2 | in-use |    2 | Attached to sanchpet_vm_2 on /dev/vda  |
| 7209fbab-6d0c-4a25-b8b5-3550f7789506 | sanchpet_disk   | in-use |    2 | Attached to sanchpet_vm_1 on /dev/vda  |
+--------------------------------------+-----------------+--------+------+----------------------------------------+
```
Проверим детали неполадки:
```bash
[root@localhost ~(keystone_admin)]# less /var/log/cinder/volume.log
<...>
2024-03-16 15:19:19.373 12803 ERROR cinder.volume.manager [req-3452e452-9993-4745-8f77-9d7ca0fca974 - - - - -] Failed to initialize driver.: cinder.exception.VolumeBackendAPIException: Bad or unexpected response from the storage volume backend API: rados and rbd python libraries not found
```
Установим пакеты для авторизации в CEPH:
```bash
[root@localhost ~(keystone_admin)]# yum install python-rbd
[root@localhost ~(keystone_admin)]# yum install ceph-common
[root@localhost ~(keystone_admin)]# yum install python-rados
```
Однако после этой ошибки возникла проблема с авторизацией:
```bash
2024-03-16 15:45:10.095 29690 ERROR cinder.volume.drivers.rbd [None req-e3624383-0238-4d55-bf12-423ed21cf239 - - - - - -] Error connecting to ceph cluster.: rados.PermissionDeniedError: [errno 13] RADOS permission denied (error connecting to the cluster)
```
Эту ошибку, к сожалению, я уже не смог побороть (не осталось времени). Скорее всего, ошибка была допущена или при создании пользователя для cinder, или при настройке `cinder.conf` на машине OpenStack, что привело к сбоям при авторизации.
### Вопросы
1. *Какие протоколы тунеллирования использует Neutron?*
   
   **Согласно [документации](https://docs.openstack.org/neutron/rocky/admin/intro-overlay-protocols.html) OpenStack Neutron использует три типа протоколов туннелирования: GRE, VXLAN и GENEVE. GRE используется для установки point-to-point подключений поверх IP. VXLAN - механизм изоляции сетей. GENEVE - фреймворк для виртуализации сетей.**
2. *Можно ли заменить Cinder, например, CEPH-ом? Для чего если да, почему если нет?*
   
   **Да, Cinder можно заменить CEPH'ом. Включить CEPH как бэкенд можно в конфигурационном файле Cinder, по умолчанию он лежит в `/etc/cinder/cinder.conf`. Подключая CEPH в качестве хранилища блочных устройств, мы получаем все преимущества S3 в сравнении с традиционными хранилищами.** 
### Заключение
В ходе работы было проведено взаимодействие с OpenStack API. Был получен токен, с помощью которого производилась авторизация и осуществлялись действия по управлению ресурсами OpenStack через эндпоинты. Также "почти успешно" был подключен CEPH в качестве бэкенда для Cinder, рассмотрена теоретическая сторона применения CEPH в качестве хранилища блочных устройств. Результаты, полученные в ходе работы, в дальнейшем могут быть использованы для управления установками OpenStack через API. 

