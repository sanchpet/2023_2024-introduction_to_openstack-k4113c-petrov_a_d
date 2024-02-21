___
# Отчёт о лабораторной работе №1

## Подготовка к развертыванию OpenStack

- [Описание работы](#описание-работы)
- [Цель работы](#цель-работы)
- [Ход работы](#ход-работы)
  * [Создание ВМ](#создание-вм)
  * [Установка OpenStack](#установка-openstack)
  * [Проверка установки OpenStack](#проверка-установки-openstack)
- [Заключение](#заключение)
### Описание работы
Это первая лабораторная работа курса, в которой предлагается познакомиться с OpenStack, не выполняя пока серьёзных работ. Потребуется создать ВМ, установить OpenStack и проверить его работспособность.
### Цель работы
Получить практические навыки установки OpenStack на сервер на базе ОС RHEL.
### Ход работы

#### Создание ВМ
С помощью гипервизора VirtualBox создана виртуальная машина на базе образа дистрибутива [Alma Linux 9.3](https://repo.almalinux.org/almalinux/9.3/isos/x86_64/AlmaLinux-9.3-x86_64-minimal.iso) со следующими параметрами:
- 4 гб RAM;
- 30 гб дискового пространства;
- 2 vCPU;
- Сетевой интерфейс в режиме «Сетевой мост».
Настроен пользователь root и разрешён вход по SSH для пользователя root. Сетевой интерфейс после установки и запуска машины получил адрес `192.168.1.56`.
#### Установка OpenStack
После входа по SSH на машину
```bash
root@DESKTOP-9FUG7NH:~# ssh root@192.168.1.56
```
Устанавливается пакет git, после чего клонируется репозиторий с конфигурационными скриптами от преподавателя
```bash
[root@localhost ~]# dnf install git -y
[root@localhost ~]# git clone https://gitlab.com/itmo_samon/openstack_lab.git
[root@localhost ~]# cd openstack_lab/
[root@localhost openstack_lab]# ls -l
total 16
-rw-r--r--. 1 root root 6185 Feb 21 14:17 README.md
-rwxr-xr-x. 1 root root  868 Feb 21 14:17 config.sh
-rwxr-xr-x. 1 root root 1142 Feb 21 14:17 prepare.sh
```
Скрипт `prepare.sh` отключает брандмауэр и снижает уровень защиты selinux, устанавливает пакеты для управления openstack, генерирует answer file, а также увеличивает системный swap, чтобы RAM на машине в сумме давала 12 гб (для гарантии корректной установки Openstack). Ниже представлено состояние памяти после установки OpenStack:
```bash
[root@localhost openstack_lab]# free -h
               total        used        free      shared  buff/cache   available
Mem:           3.6Gi       2.8Gi       391Mi        39Mi       759Mi       794Mi
Swap:          8.4Gi        28Mi       8.4Gi
```
Скрипт config.sh вносит изменения в answer file, отключая некоторые модули и понижая настройки параметров, отвечающих за потребление ресурсов. После последовательного запуска двух скриптов выполняется установка OpenStack через `packstack` – утилиту, которая с помощью puppet устанавливает на сервер нужные части OpenStack, беря за основу настройки из answer файла.
```bash
[root@localhost openstack_lab]# packstack --answer-file=answer.cfg
```
Скрипт выполняется долго, пока затягиваются настройки из puppet. На моей машине установка длилась 18 минут. Получаем заветный вывод об успешности установки и приступаем к следующему этапу работы.
```
**** Installation completed successfully ******
```
#### Проверка установки OpenStack
Проверим файл настроек Keystone, содержащий лог-пассы для входа, имя региона, URL аутентификации, приглашение оболочки, настройки проект и версию API:
```bash
[root@localhost ~]# cat ./keystonerc_admin
unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD='f1d2da99ad3d4dac'
    export OS_REGION_NAME=RegionOne
    export OS_AUTH_URL=http://192.168.1.56:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
```
Экспортируем переменные оболочки и видим, как обновилось приглашение, указывая, что окружение теперь настроено для управления Keystone:
```bash
[root@localhost ~]# source ./keystonerc_admin
[root@localhost ~(keystone_admin)]#
```
Проверим все эндпоинты OpenStack, выполнив команду `openstack endpoint list`. 
```
[root@localhost ~(keystone_admin)]# openstack endpoint list
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------              +
| ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                         |
+----------------------------------+-----------+--------------+--------------+---------+-----------+-------------------------------              +
| 046b2d6fa5fc4ce3aff4c2e8760c6eb1 | RegionOne | placement    | placement    | True    | public    | http://192.168.1.56:8778                    |
| 215c451f6c3241b181819c42d4b8dd17 | RegionOne | glance       | image        | True    | admin     | http://192.168.1.56:9292                    |
| 478a9794f28e4865a28cb39f56ce426e | RegionOne | placement    | placement    | True    | internal  | http://192.168.1.56:8778                    |
| 4acfed592a894b6991c3b74ccc157ade | RegionOne | glance       | image        | True    | public    | http://192.168.1.56:9292                    |
| 4d8af940289d4e96883a93b7098e8794 | RegionOne | keystone     | identity     | True    | internal  | http://192.168.1.56:5000                    |
| 565106c57da04da5bfa97980810f78ad | RegionOne | cinderv3     | volumev3     | True    | public    | http://192.168.1.56:8776/v3                 |
| 5b13f7e5b6514901be1f3a2fbde82f08 | RegionOne | nova         | compute      | True    | public    | http://192.168.1.56:8774/v2.1               |
| 65a137886ce348c6b2fcb94a94c97974 | RegionOne | neutron      | network      | True    | admin     | http://192.168.1.56:9696                    |
| 80fb8a3e142b4dd0828a418bc03c8654 | RegionOne | keystone     | identity     | True    | public    | http://192.168.1.56:5000                    |
| 830feae9eb3b441981009a0992815759 | RegionOne | cinderv3     | volumev3     | True    | internal  | http://192.168.1.56:8776/v3                 |
| 84d7a795026d4a4cb840bff9fb5678a5 | RegionOne | neutron      | network      | True    | public    | http://192.168.1.56:9696                    |
| 96a7e0c7e0ce4815ade0ea10e6af0b61 | RegionOne | neutron      | network      | True    | internal  | http://192.168.1.56:9696                    |
| 97a7003e2f364362a60da606503ed838 | RegionOne | nova         | compute      | True    | admin     | http://192.168.1.56:8774/v2.1               |
| bd20c7a4047b41d4b575a231c3162b43 | RegionOne | glance       | image        | True    | internal  | http://192.168.1.56:9292                    |
| c6beedcbb0cb498d992861271d68a756 | RegionOne | cinderv3     | volumev3     | True    | admin     | http://192.168.1.56:8776/v3                 |
| c8d329dc25cf427b8a095a6fb7390185 | RegionOne | placement    | placement    | True    | admin     | http://192.168.1.56:8778                    |
| d3771005b2054a7fa59771fb19800735 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.1.56:5000                    |
| e03d2dc80c82426280a0f8f7ec4dd62e | RegionOne | nova         | compute      | True    | internal  | http://192.168.1.56:8774/v2.1               |
+----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------- 
```
Все сервисы относятся к одному региону, при этом каждый сервис имеет по несколько эндпоинтов. Эндопинты одних и тех же сервисов отличаются типами интерфейсов, но на данном этапе URL для доступа к ним совпадают. Проверим список всех пользователей OpenStack:
```bash
[root@localhost ~(keystone_admin)]# openstack user list
+----------------------------------+-----------+
| ID                               | Name      |
+----------------------------------+-----------+
| a5b8b6dd7ffa47eabfda49c8203ecafe | admin     |
| 13068e55ecb24e4586236207fcb21692 | glance    |
| e6f29e4a81d549c59a50b7a843eb4c17 | cinder    |
| 51b71c142c0c4987909258b22fdb08da | nova      |
| 4a1bfae008d6423dbf434724fab65e0a | placement |
| 49c9abd3446b45bca1d6e159b87c29c1 | neutron   |
+----------------------------------+-----------+
```
Есть административный пользователь, а также пользователи для некоторых сервисов. Проверим список проектов:
```bash
[root@localhost ~(keystone_admin)]# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 2dd20895306a45ac9418cf988a170262 | admin    |
| d90b1e421381499cb63cac5983373e35 | services |
+----------------------------------+----------+
```
Есть два проекта, административный и сервисный. Создадим новый проект в домене по умолчанию и убедимся, что он отображается в списке:
```bash
[root@localhost ~(keystone_admin)]# openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | b973a508aa6641678e6dd6fb3c024deb |
| is_domain   | False                            |
| name        | demo                             |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
[root@localhost ~(keystone_admin)]# openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 2dd20895306a45ac9418cf988a170262 | admin    |
| b973a508aa6641678e6dd6fb3c024deb | demo     |
| d90b1e421381499cb63cac5983373e35 | services |
+----------------------------------+----------+
```
Проверим, слушает ли Apache порт 80, и авторизуемся в панели OpenStack по адресу [http://192.168.1.56:80](http://192.168.1.56:80).
```bash
[root@localhost ~(keystone_admin)]# ss -tunlp | grep :80
tcp   LISTEN 0      511                *:80               *:*    users:(("httpd",pid=47910,fd=9),("httpd",pid=47909,fd=9),("httpd",pid=47908,fd=9),("httpd",pid=47871,fd=9),("httpd",pid=46394,fd=9),("httpd",pid=46393,fd=9),("httpd",pid=46392,fd=9),("httpd",pid=46391,fd=9),("httpd",pid=46390,fd=9),("httpd",pid=46389,fd=9),("httpd",pid=46388,fd=9),("httpd",pid=46387,fd=9),("httpd",pid=46380,fd=9))
```
После авторизации попадаем на главную страницу OpenStack.
![img1](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020240221175854.png)
Создадим новый проект.
![img2](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020240221175903.png)
И добавим пользователя в проект с ролью «\_member\_».
![img3](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020240221175908.png)
Авторизуемся под новым пользователем `sanchpet` и убедимся, что ему доступен только один проект, на который мы его добавили.
![img4](https://github.com/sanchpet/2023_2024-introduction_to_openstack-k4113c-petrov_a_d/blob/main/lab1/img/Pasted%20image%2020240221175915.png)
### Заключение
В ходе работы было выполнено создание виртуальной машины, произведена установка OpenStack на виртуальную машину, проверена работоспособность OpenStack. В ходе работы получено представление об эндпоинтах, пользователях и проектах OpenStack, изучены возможности OpenStack CLI, произведен вход в OpenStack при помощи Horizon. Полученные результаты в дальнейшем могут быть использованы для настройки локальных лабораторных окружений для проведения работ в OpenStack. Задачи работы выполнены, цель достигнута.
