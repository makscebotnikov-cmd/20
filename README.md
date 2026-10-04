# Домашнее задание к занятию «Кластеры. Ресурсы под управлением облачных провайдеров»

## ` Чеботников М.Б.`

---

## Цели задания
### 1. Организация кластера Kubernetes и кластера баз данных MySQL в отказоустойчивой архитектуре.
### 2. Размещение в private подсетях кластера БД, а в public — кластера Kubernetes.

---

## Задание 1. Yandex Cloud

### 1. Настроить с помощью Terraform кластер баз данных MySQL.
  * Используя настройки VPC из предыдущих домашних заданий, добавить дополнительно подсеть private в разных зонах, чтобы       обеспечить отказоустойчивость.
  * Разместить ноды кластера MySQL в разных подсетях.
  * Необходимо предусмотреть репликацию с произвольным временем технического обслуживания.
  * Использовать окружение Prestable, платформу Intel Broadwell с производительностью 50% CPU и размером диска 20 Гб.
  * Задать время начала резервного копирования — 23:59.
  * Включить защиту кластера от непреднамеренного удаления.
  * Создать БД с именем netology_db, логином и паролем.

### 2. Настроить с помощью Terraform кластер Kubernetes.
  * Используя настройки VPC из предыдущих домашних заданий, добавить дополнительно две подсети public в разных зонах,          чтобы обеспечить отказоустойчивость.
  * Создать отдельный сервис-аккаунт с необходимыми правами.
  * Создать региональный мастер Kubernetes с размещением нод в трёх разных подсетях.
  * Добавить возможность шифрования ключом из KMS, созданным в предыдущем домашнем задании.
  * Создать группу узлов, состояющую из трёх машин с автомасштабированием до шести.
  * Подключиться к кластеру с помощью kubectl.
  * *Запустить микросервис phpmyadmin и подключиться к ранее созданной БД.
  * *Создать сервис-типы Load Balancer и подключиться к phpmyadmin. Предоставить скриншот с публичным адресом и                подключением к БД.

---

## Ответ:

### 1. Файл `variables.tf`

Определение переменных для гибкой настройки облака.

```Hcl

variable "yc_token" {}
variable "yc_cloud_id" {}
variable "yc_folder_id" {}
variable "region" { default = "ru-central1" }
variable "mysql_user_password" { default = "Netology123!" }
variable "zones" {
  default = ["ru-central1-a", "ru-central1-b", "ru-central1-d"]
}
```

---

### 2. Файл `vpc.tf`

Настройка сети, подсетей (приватных для БД и публичных для K8s), KMS ключа и NAT-шлюза.

```Hcl
resource "yandex_vpc_network" "vpc" {
  name = "netology-vpc"
}

resource "yandex_kms_symmetric_key" "mysql_kms_key" {
  name              = "netology-key"
  default_algorithm = "AES_256"
}

resource "yandex_iam_service_account" "k8s_sa" {
  name = "sa-k8s-netology"
}

resource "yandex_resourcemanager_folder_iam_member" "k8s_roles" {
  for_each = toset([
    "k8s.clusters.agent",
    "vpc.publicAdmin",
    "container-registry.images.puller",
    "kms.viewer",
    "kms.keys.decrypter",
    "load-balancer.admin",
    "editor"
  ])
  folder_id = var.yc_folder_id
  role      = each.key
  member    = "serviceAccount:${yandex_iam_service_account.k8s_sa.id}"
}

resource "yandex_vpc_gateway" "nat_gateway" {
  name = "nat-gateway"
  shared_egress_gateway {}
}

resource "yandex_vpc_route_table" "nat_route_table" {
  network_id = yandex_vpc_network.vpc.id
  static_route {
    destination_prefix = "0.0.0.0/0"
    gateway_id         = yandex_vpc_gateway.nat_gateway.id
  }
}

resource "yandex_vpc_subnet" "mysql_private_1" {
  name           = "mysql-private-a"
  zone           = var.zones[0]
  v4_cidr_blocks = ["10.10.1.0/24"]
  network_id     = yandex_vpc_network.vpc.id
  route_table_id = yandex_vpc_route_table.nat_route_table.id
}

resource "yandex_vpc_subnet" "mysql_private_2" {
  name           = "mysql-private-b"
  zone           = var.zones[1]
  v4_cidr_blocks = ["10.10.2.0/24"]
  network_id     = yandex_vpc_network.vpc.id
  route_table_id = yandex_vpc_route_table.nat_route_table.id
}

resource "yandex_vpc_subnet" "k8s_public_1" {
  name           = "k8s-public-a"
  zone           = var.zones[0]
  v4_cidr_blocks = ["10.10.10.0/24"]
  network_id     = yandex_vpc_network.vpc.id
}

resource "yandex_vpc_subnet" "k8s_public_2" {
  name           = "k8s-public-b"
  zone           = var.zones[1]
  v4_cidr_blocks = ["10.10.11.0/24"]
  network_id     = yandex_vpc_network.vpc.id
}

resource "yandex_vpc_subnet" "k8s_public_3" {
  name           = "k8s-public-d"
  zone           = var.zones[2]
  v4_cidr_blocks = ["10.10.12.0/24"]
  network_id     = yandex_vpc_network.vpc.id
}
```

---

### 3. Файл `mysql.tf`

Создание кластера MySQL в двух зонах.

```Hcl
resource "yandex_mdb_mysql_cluster" "mysql" {
  name        = "netology-mysql"
  environment = "PRESTABLE"
  network_id  = yandex_vpc_network.vpc.id
  version     = "8.0"

  resources {
    resource_preset_id = "s2.micro" # Broadwell, 50% CPU
    disk_type_id       = "network-ssd"
    disk_size          = 20
  }

  host {
    zone      = var.zones[0]
    subnet_id = yandex_vpc_subnet.mysql_private_1.id
  }
  host {
    zone      = var.zones[1]
    subnet_id = yandex_vpc_subnet.mysql_private_2.id
  }

  backup_window_start {
    hours   = 23
    minutes = 59
  }

  encryption {
    kms_key_id = yandex_kms_symmetric_key.mysql_kms_key.id
  }

  deletion_protection = true
}

resource "yandex_mdb_mysql_database" "db" {
  cluster_id = yandex_mdb_mysql_cluster.mysql.id
  name       = "netology_db"
}

resource "yandex_mdb_mysql_user" "db_user" {
  cluster_id = yandex_mdb_mysql_cluster.mysql.id
  name       = "netology_user"
  password   = var.mysql_user_password
}
```

---

### 4. Файл `k8s.tf`

Создание регионального мастера Kubernetes и групп узлов с автомасштабированием.

```Hcl
resource "yandex_kubernetes_cluster" "k8s_regional" {
  name = "k8s-netology"
  network_id = yandex_vpc_network.vpc.id

  master {
    regional {
      region = "ru-central1"
      location {
        zone      = yandex_vpc_subnet.k8s_public_1.zone
        subnet_id = yandex_vpc_subnet.k8s_public_1.id
      }
      location {
        zone      = yandex_vpc_subnet.k8s_public_2.zone
        subnet_id = yandex_vpc_subnet.k8s_public_2.id
      }
      location {
        zone      = yandex_vpc_subnet.k8s_public_3.zone
        subnet_id = yandex_vpc_subnet.k8s_public_3.id
      }
    }
    public_ip = true
  }

  service_account_id      = yandex_iam_service_account.k8s_sa.id
  node_service_account_id = yandex_iam_service_account.k8s_sa.id

  kms_provider {
    key_id = yandex_kms_symmetric_key.mysql_kms_key.id
  }
}

resource "yandex_kubernetes_node_group" "node_groups" {
  for_each = {
    "a" = yandex_vpc_subnet.k8s_public_1.id
    "b" = yandex_vpc_subnet.k8s_public_2.id
    "d" = yandex_vpc_subnet.k8s_public_3.id
  }

  cluster_id = yandex_kubernetes_cluster.k8s_regional.id
  name       = "node-group-${each.key}"

  instance_template {
    platform_id = "standard-v2" # Intel Broadwell
    network_interface {
      nat = true
      subnet_ids = [each.value]
    }
    resources {
      memory = 4
      cores  = 2
      core_fraction = 50
    }
    boot_disk {
      type = "network-ssd"
      size = 32
    }
  }
  scale_policy {
    auto_scale {
      min     = 1
      max     = 2
      initial = 1
    }
  }
  allocation_policy {
    location { zone = "ru-central1-${each.key}" }
  }
}
```

---

### 5. Файл phpmyadmin.yaml

Манифест Kubernetes для развертывания приложения.

```Yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: phpmyadmin/phpmyadmin
        ports:
        - containerPort: 80
        env:
        - name: PMA_HOST
          value: "rc1a-xxxxxxxx.mdb.yandexcloud.net" # FQDN из консоли MySQL
        - name: PMA_USER
          value: "netology_user"
        - name: PMA_PASSWORD
          value: "Netology123!"
---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: phpmyadmin
```

---

## Итоги выполнения

---

<img width="1211" height="509" alt="1" src="https://github.com/user-attachments/assets/66c26ae1-9a8e-48da-bf43-14c13efa91a0" />


---

### 1. MySQL: Настроен кластер из 2-х хостов (Master + Replicas) в окружении PRESTABLE. Включена защита от удаления и        шифрование дисков KMS. Время бэкапа — 23:59.

<img width="1491" height="410" alt="2" src="https://github.com/user-attachments/assets/e9f7404c-3106-49ec-bb84-d467ff7ee312" />



---

### 2. Kubernetes: Создан региональный мастер. Ноды распределены по 3-м зонам (a, b, d). Включено шифрование секретов ключом KMS. Настроен автоскейлинг (суммарно 3-6 нод).


<img width="827" height="442" alt="3" src="https://github.com/user-attachments/assets/3f8807ae-0bf5-4adb-8575-693056274437" />


---

### 3. Доступ: Подключение к кластеру выполнено через kubectl. Сервис phpMyAdmin запущен и доступен через внешний Load Balancer. База данных успешно подключена.

---


<img width="1256" height="202" alt="4" src="https://github.com/user-attachments/assets/123d7037-4f1d-45e7-ab07-4cc597d39996" />


---

<img width="1614" height="881" alt="5" src="https://github.com/user-attachments/assets/5e339585-d7e2-4f89-911c-1ffc6e9b6e8f" />


---

<img width="1190" height="965" alt="6" src="https://github.com/user-attachments/assets/4e547727-1ce1-4619-be03-5361e59f9859" />


---









