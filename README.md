
## 1.0 Установка и подготовка Terraform и Ansible.
### 1.1 Установка и подготовка Terraform.


Скачиваю и распаковываю последнюю стабильную версию на 22.05.2025 с сайта: https://hashicorp-releases.yandexcloud.net/terraform/
```bash
wget https://hashicorp-releases.yandexcloud.net/terraform/1.12.1/terraform_1.12.1_linux_amd64.zip

unzip terraform_1.12.1_linux_amd64.zip
chmod 744 terraform
mv terraform /usr/local/bin/
terraform -version
```
![изображение](https://github.com/user-attachments/assets/fbe22e3f-4d72-4cb0-9fb6-743ac247b6ff)

Создаю файл `.terraformrc` и добавляю блок с источником, из которого будет устанавливаться провайдер.
```bash
nano ~/.terraformrc
```
```terraform
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
```
![изображение](https://github.com/user-attachments/assets/fc32a377-800a-42d5-a1cf-d09328f32036)



Для файла с метаданными, `meta.yaml`, необходим публичный SSH-ключ для доступа к ВМ. Для Yandex Cloud рекомендуется использовать алгоритм Ed25519: сгенерированные по нему ключи — самые безопасные. Ссылка: https://cloud.yandex.ru/ru/docs/glossary/ssh-keygen
```bash
ssh-keygen -t ed25519
```
![изображение](https://github.com/user-attachments/assets/4d2d611e-5d9a-4542-902e-af9bd72de964)



Создаю файл `meta.yaml` с данными пользователя на создаваемые ВМ.
```bash
nano ~/meta.yaml
```
```terraform
#cloud-config
 users:
  - name: tverdyakov
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ssh-ed25519
```
![изображение](https://github.com/user-attachments/assets/00d4821e-6844-490f-8493-2466eaea3885)

Создаю `playbook Terraform` c блоком провайдера.
```bash
nano ~/main.tf
```
```terraform
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}
```
![изображение](https://github.com/user-attachments/assets/90603707-e4e5-4bf8-a25f-532c447200b3)

Инициализирую провайдера.
```bash
terraform-diplom init
```
![изображение](https://github.com/user-attachments/assets/0ae59a28-7041-4037-8205-242c82620351)

 Настройка доступа к Yandex Cloud

    curl https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
    exec -l $SHELL
    yc init
Получил идентификаторы

    yc config list | grep -E 'cloud-id|folder-id'


![изображение](https://github.com/user-attachments/assets/7f34f022-42a5-4962-ae2b-1d752b33e052)




---


### 1.2 Заполнение конфигурационного файла terraform `main.tf` для выполнения задач дипломной работы.

Ссылки на файлы terraform: 

[main.tf](main.tf.txt)

[meta.yaml]

[.terraformrc]

#### По условиям задачи необходимо развернуть через terraform следующий ресурcы:

##### Сайт. Веб-сервера. Nginx.
- Создать две ВМ в разных зонах, установить на них сервер nginx.
- Создать Target Group, включить в неё две созданные ВМ.
- Создать Backend Group, настроить backends на target group, ранее созданную. Настроить healthcheck на корень (/) и порт 80, протокол HTTP.
- Создать HTTP router. Путь указать — /, backend group — созданную ранее.
- Создать Application load balancer для распределения трафика на веб-сервера, созданные ранее. Указать HTTP router, созданный ранее, задать listener тип auto, порт 80.
```terraform
####################
## Nginx-web-1 #####
####################
resource "yandex_compute_instance" "nginx-web-1" {
  name  = "nginx-web-1"
  hostname = "nginx-web-1"
  zone  = yandex_vpc_subnet.a-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.10.3"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.nginx-web-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

####################
## Nginx-web-2 #####
####################
resource "yandex_compute_instance" "nginx-web-2" {
  name  = "nginx-web-2"
  hostname = "nginx-web-2"
  zone  = yandex_vpc_subnet.b-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.b-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.20.3"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.nginx-web-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

#################################################################################################################
## Target group ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/target-group-create ##
#################################################################################################################
resource "yandex_alb_target_group" "nginx-target-group"{
  name = "nginx-target-group"

  target {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ip_address   = "${yandex_compute_instance.nginx-web-1.network_interface.0.ip_address}"
  } 

  target {
    subnet_id = "${yandex_vpc_subnet.b-subnet-diplom.id}"
    ip_address   = "${yandex_compute_instance.nginx-web-2.network_interface.0.ip_address}"
  }  
}

###################################################################################################################
## Backend group ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/backend-group-create ##
###################################################################################################################
resource "yandex_alb_backend_group" "nginx-backend-group" {
  name = "nginx-backend-group"
  session_affinity {
    connection {
      source_ip = false
    }
  }

  http_backend {
    name                   = "http-backend"
    weight                 = 1
    port                   = 80
    target_group_ids       = ["${yandex_alb_target_group.nginx-target-group.id}"]
    load_balancing_config {
      panic_threshold      = 90
    }    
    healthcheck {
      timeout              = "10s"
      interval             = "2s"
      healthy_threshold    = 10
      unhealthy_threshold  = 15 
      http_healthcheck {
        path               = "/"
      }
    }
  }
}

###############################################################################################################
## HTTP router ##### https://cloud.yandex.ru/ru/docs/application-load-balancer/operations/http-router-create ##
###############################################################################################################
resource "yandex_alb_http_router" "nginx-tf-router" {
  name   = "nginx-tf-router"
  labels = {
    tf-label    = "tf-label-value"
    empty-label = ""
  }
}

resource "yandex_alb_virtual_host" "nginx-virtual-host" {
  name           = "nginx-virtual-host"
  http_router_id = yandex_alb_http_router.nginx-tf-router.id
  route {
    name = "nginx-route"
    http_route {
      http_route_action {
        backend_group_id = yandex_alb_backend_group.nginx-backend-group.id
        timeout          = "60s"
      }
    }
  }
}    

############################################################################################################################################
## Application load balancer ##### https://cloud.yandex.com/ru/docs/application-load-balancer/operations/application-load-balancer-create ##
############################################################################################################################################
resource "yandex_alb_load_balancer" "nginx-balancer" {
  name        = "nginx-balancer"
  network_id  = "${yandex_vpc_network.network-diplom.id}"

  allocation_policy {
    location {
      zone_id   = "${yandex_vpc_subnet.c-subnet-diplom.zone}"
      subnet_id = yandex_vpc_subnet.c-subnet-diplom.id
    }
  }

  listener {
    name = "nginx-listener"
    endpoint {
      address {
        external_ipv4_address {
        }
      }
      ports = [ 80 ]
    }
    http {
      handler {
        http_router_id = yandex_alb_http_router.nginx-tf-router.id
      }
    }
  }
}
```

##### Мониторинг. Zabbix. Zabbix-agent.
- Создать ВМ, развернуть на ней Zabbix. На каждую ВМ установить Zabbix Agent, настроить агенты на отправление метрик в Zabbix.
```terraform
###############
## Zabbix #####
###############
resource "yandex_compute_instance" "zabbix" {
  name  = "zabbix"
  hostname = "zabbix"
  zone  = yandex_vpc_subnet.c-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.c-subnet-diplom.id}"
    nat = true
    ipv4 = true
    ip_address = "192.168.30.4"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.zabbix-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}
```

##### Логи. Elasticsearch. Kibana. Filebeat.
- Cоздать ВМ, развернуть на ней Elasticsearch. Установить Filebeat в ВМ к веб-серверам, настроить на отправку access.log, error.log nginx в Elasticsearch.
- Создать ВМ, развернуть на ней Kibana, сконфигурировать соединение с Elasticsearch.
```terraform
######################
## Elasticsearch #####
######################
resource "yandex_compute_instance" "elasticsearch" {
  name  = "elasticsearch"
  hostname = "elasticsearch"
  zone  = yandex_vpc_subnet.a-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 4
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.a-subnet-diplom.id}"
    ipv4 = true
    ip_address = "192.168.10.4"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.elasticsearch-security.id, yandex_vpc_security_group.kibana-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

###############
## Kibana #####
###############
resource "yandex_compute_instance" "kibana" {
  name  = "kibana"
  hostname = "kibana"
  zone  = yandex_vpc_subnet.c-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.c-subnet-diplom.id}"
    nat = true
    ipv4 = true
    ip_address = "192.168.30.5"
    security_group_ids = [yandex_vpc_security_group.bastion-security-local.id, yandex_vpc_security_group.elasticsearch-security.id, yandex_vpc_security_group.kibana-security.id, yandex_vpc_security_group.filebeat-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}
```

##### Сеть.
- Развернуть один VPC.
- Сервера web, Elasticsearch поместить в приватные подсети. 
- Сервера Zabbix, Kibana, application load balancer определить в публичную подсеть.
- Настроить Security Groups соответствующих сервисов на входящий трафик только к нужным портам.
- Настроить ВМ с публичным адресом, в которой будет открыт только один порт — ssh. Эта вм будет реализовывать концепцию bastion host.
```terraform
#################################################################################
## Network ##### https://cloud.yandex.ru/ru/docs/vpc/operations/network-create ##
#################################################################################
resource "yandex_vpc_network" "network-diplom" {
  name        = "network-diplom"
  description = "Network diplom"
}

###########################################################################################################
## Subnet. Gateway. Route table ###### https://cloud.yandex.ru/ru/docs/vpc/operations/create-nat-gateway ##
###########################################################################################################
resource "yandex_vpc_subnet" "a-subnet-diplom" {
  name           = "a-subnet-diplom"
  v4_cidr_blocks = ["192.168.10.0/24"]
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.network-diplom.id
  route_table_id = yandex_vpc_route_table.a-b-subnet-route-table.id
}

resource "yandex_vpc_subnet" "b-subnet-diplom" {
  name           = "b-subnet-diplom"
  v4_cidr_blocks = ["192.168.20.0/24"]
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.network-diplom.id
  route_table_id = yandex_vpc_route_table.a-b-subnet-route-table.id
}

resource "yandex_vpc_subnet" "c-subnet-diplom" {
  name           = "c-subnet-diplom"
  v4_cidr_blocks = ["192.168.30.0/24"]
  zone           = "ru-central1-c"
  network_id     = yandex_vpc_network.network-diplom.id
}

resource "yandex_vpc_gateway" "gateway-route-table" {
  name = "gateway-route-table"
  shared_egress_gateway {}
}

resource "yandex_vpc_route_table" "a-b-subnet-route-table" {
  name           = "a-b-subnet-route-table"
  network_id     = yandex_vpc_network.network-diplom.id

  static_route {
    destination_prefix = "0.0.0.0/0"
    gateway_id         = yandex_vpc_gateway.gateway-route-table.id
  }
}

########################################################################################
## Security_groups ##### https://cloud.yandex.ru/ru/docs/vpc/concepts/security-groups ##
########################################################################################
resource "yandex_vpc_security_group" "bastion-security-local" {
  name        = "bastion-security-local"
  description = "Bastion security for local ip"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "TCP"
    description    = "IN to 22 port from local ip"
    v4_cidr_blocks = ["192.168.10.0/24", "192.168.20.0/24", "192.168.30.0/24"]
    port           = 22
  }

  egress {
    protocol       = "TCP"
    description    = "OUT from 22 port to local ip"
    v4_cidr_blocks = ["192.168.10.0/24", "192.168.20.0/24", "192.168.30.0/24"]
    port           = 22
  } 

  egress {
    protocol       = "ANY"
    description    = "OUT from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    from_port      = 0
    to_port        = 65535
  }
}

resource "yandex_vpc_security_group" "bastion-security" {
  name        = "bastion-security"
  description = "Bastion security to connect to bastion"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol          = "TCP"
    description       = "IN to 22 port from any ip"
    v4_cidr_blocks    = ["0.0.0.0/0"]
    port              = 22
  }

  egress {
    protocol          = "ANY"
    description       = "OUT from any ip"
    v4_cidr_blocks    = ["0.0.0.0/0"]
    from_port         = 0
    to_port           = 65535
  }  

  ingress {
    protocol          = "TCP"
    description       = "IN to 22 port from local ip"
    security_group_id = yandex_vpc_security_group.bastion-security-local.id
    port              = 22
  }

  egress {
    protocol          = "TCP"
    description       = "OUT from 22 port to local ip"
    security_group_id = yandex_vpc_security_group.bastion-security-local.id
    port              = 22
  }
}

resource "yandex_vpc_security_group" "nginx-web-security" {
  name        = "nginx-web-security"
  description = "Nginx-web security"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "ANY"
    description    = "IN to 80 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  egress {
    protocol       = "ANY"
    description    = "OUT from 80 port to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  ingress {
    protocol       = "ANY"
    description    = "IN to 10050 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }

  egress {
    protocol       = "ANY"
    description    = "OUT from 10050 port to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }
}

resource "yandex_vpc_security_group" "zabbix-security" {
  name        = "zabbix-security"
  description = "Zabbix security"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "TCP"
    description    = "IN to 80 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  egress {
    protocol       = "TCP"
    description    = "OUT from 80 port to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 80
  }

  ingress {
    protocol       = "TCP"
    description    = "IN to 10051 from local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 10051
  }

    egress {
    protocol       = "TCP"
    description    = "OUT from 10051 port to local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 10051
  }
}

resource "yandex_vpc_security_group" "elasticsearch-security" {
  name        = "elasticsearch-security"
  description = "Elasticsearch security"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "TCP"
    description    = "IN to 9200 port from local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 9200
  }

  egress {
    protocol       = "TCP"
    description    = "OUT from 9200 port to local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 9200
  }

  ingress {
    protocol       = "ANY"
    description    = "IN to 10050 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }

  egress {
    protocol       = "ANY"
    description    = "OUT from 10050 port to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }
}

resource "yandex_vpc_security_group" "kibana-security" {
  name        = "kibana-security"
  description = "Kibana security"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "ANY"
    description    = "IN to 10050 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }

  egress {
    protocol       = "ANY"
    description    = "OUT from 10050 to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 10050
  }

  ingress {
    protocol       = "TCP"
    description    = "IN to 5601 port from any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 5601
  }

  egress {
    protocol       = "TCP"
    description    = "OUT from 5601 to any ip"
    v4_cidr_blocks = ["0.0.0.0/0"]
    port           = 5601
  }
}

resource "yandex_vpc_security_group" "filebeat-security" {
  name        = "filebeat-security"
  description = "Filebeat security"
  network_id  = yandex_vpc_network.network-diplom.id

  ingress {
    protocol       = "TCP"
    description    = "IN to 5044 port from local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 5044
  }

  egress {
    protocol       = "TCP"
    description    = "OUT from 5044 to local ip"
    v4_cidr_blocks = ["192.168.10.0/24","192.168.20.0/24","192.168.30.0/24"]
    port           = 5044
  }
}

################
## Bastion #####
################
resource "yandex_compute_instance" "bastion" {
  name  = "bastion"
  hostname = "bastion"
  zone  = yandex_vpc_subnet.c-subnet-diplom.zone
  platform_id     = "standard-v3"
  resources {
    cores         = 2
    core_fraction = 20
    memory        = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8mg6q8ujdpfh58k2bk"
      size     = 10
    }
  }

  network_interface {
    subnet_id = "${yandex_vpc_subnet.c-subnet-diplom.id}"
    nat = true
    ipv4 = true
    ip_address = "192.168.30.3"
    security_group_ids = [yandex_vpc_security_group.bastion-security.id]
  }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}
```

##### Резервное копирование.
- Создать snapshot дисков всех ВМ. 
- Ограничить время жизни snaphot в неделю. 
- Сами snaphot настроить на ежедневное копирование.
```terraform
#################################################################################################################
## Snapshot_schedule ##### https://cloud.yandex.ru/ru/docs/compute/operations/snapshot-control/create-schedule ##
#################################################################################################################
resource "yandex_compute_snapshot_schedule" "snapshot-diplom" {
  name = "snapshot-diplom"

  schedule_policy {
    expression = "30 1 * * *"
  }

  snapshot_count = 7

  snapshot_spec {
      description = "Snapshots. Every day at 01:30"
  }

  disk_ids = ["${yandex_compute_instance.bastion.boot_disk.0.disk_id}", "${yandex_compute_instance.nginx-web-1.boot_disk.0.disk_id}", "${yandex_compute_instance.nginx-web-2.boot_disk.0.disk_id}", "${yandex_compute_instance.zabbix.boot_disk.0.disk_id}", "${yandex_compute_instance.elasticsearch.boot_disk.0.disk_id}", "${yandex_compute_instance.kibana.boot_disk.0.disk_id}"]
}
```

### 2.2 Запуск terraform playbook.
```bash
terraform-diplom apply
```
![изображение](https://github.com/user-attachments/assets/81c23d01-d2d9-4ba8-a635-75ce52e51342)


---

### 2.3 Проверка развернутых ресурсов в Yandex Cloud.


---
### 1.2 Установка и подготовка Ansible.
Устанавливаю Ansible и проверяю версию.
```bash
apt install ansible
ansible --version
```
![изображение](https://github.com/user-attachments/assets/f4974bc7-56a2-4614-ae76-eb1c3a59b67c)

Создаю полностью прокомментированный пример `ansible.cfg` и заменяю содержимое файла на необходимые опции. Файл прикреплю в основной части.
```bash
ansible-config init --disabled > ansible.cfg
nano ~/ansible.cfg
```
![](https://github.com/tverdyakov/diplom_tverdyakov-sys-20/blob/main/01_Установка%20и%20подготовка%20Terraform%20и%20Ansible/screenshots/08.png)
Создаю файл `hosts` и добавляю в него начальные данные. Файл прикреплю в основной части.
```bash
nano ~/hosts
```
#### Ansible готов к использованию. Продолжение в основной части.

[Ссылка на основную часть дипломной работы.](https://github.com/tverdyakov/diplom_tverdyakov-sys-20/blob/main/02_Основная%20часть%20дипломной%20работы/README.md)
---
