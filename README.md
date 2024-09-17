# Домашнее задание к занятию «Отказоустойчивость в облаке» - `Александра Бужор`

## Задание 1

Возьмите за основу решение к заданию 1 из занятия «Подъём инфраструктуры в Яндекс Облаке».

Теперь вместо одной виртуальной машины сделайте terraform playbook, который:
создаст 2 идентичные виртуальные машины. Используйте аргумент count для создания таких ресурсов;
создаст таргет-группу. Поместите в неё созданные на шаге 1 виртуальные машины;
создаст сетевой балансировщик нагрузки, который слушает на порту 80, отправляет трафик на порт 80 виртуальных машин и http healthcheck на порт 80 виртуальных машин.
Рекомендуем изучить документацию сетевого балансировщика нагрузки для того, чтобы было понятно, что вы сделали.

Установите на созданные виртуальные машины пакет Nginx любым удобным способом и запустите Nginx веб-сервер на порту 80.

Перейдите в веб-консоль Yandex Cloud и убедитесь, что:

созданный балансировщик находится в статусе Active,
обе виртуальные машины в целевой группе находятся в состоянии healthy.
Сделайте запрос на 80 порт на внешний IP-адрес балансировщика и убедитесь, что вы получаете ответ в виде дефолтной страницы Nginx.
В качестве результата пришлите:

1. Terraform Playbook.

2. Скриншот статуса балансировщика и целевой группы.

3. Скриншот страницы, которая открылась при запросе IP-адреса балансировщика.

## Ответ:

terraform playbook:

```
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
}

provider "yandex" {
  token     = "******************************************************"
  cloud_id  = "b1gvot01bgvfnc8fg65p"
  folder_id = "b1gu4fiv14lj0if4qmvn"
  zone      = "ru-central1-b"
}

resource "yandex_compute_instance" "vm" {

  count = 2
  name        = "vm${count.index}"
  platform_id = "standard-v3"
  zone        = "ru-central1-b"

  resources {
    cores  = 4
    memory = 8
  }

  boot_disk {
    initialize_params {
      image_id = "fd8auu58m9ic4rtekngm"
      size = 15
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    nat       = true
    }
    placement_policy {
      placement_group_id = "${yandex_compute_placement_group.group1.id}"
    }

  metadata = {
    user-data = "${file("./meta.yaml")}"
  }
}

resource "yandex_lb_target_group" "testtg-1" {
  name      = "testtg1"
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[0].network_interface.0.ip_address
  }
  target {
    subnet_id = yandex_vpc_subnet.subnet-1.id
    address   = yandex_compute_instance.vm[1].network_interface.0.ip_address
  }
}

resource "yandex_lb_network_load_balancer" "netotest-1" {
  name = "netotest1"
  listener {
    name = "my-list"
    port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }
  attached_target_group {
    target_group_id = yandex_lb_target_group.testtg-1.id
    healthcheck {
      name = "http"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

resource "yandex_vpc_network" "network-1" {
  name = "network1"
}

resource "yandex_vpc_subnet" "subnet-1" {
  name           = "subnet1"
  zone           = "ru-central1-b"
  v4_cidr_blocks = ["192.168.10.0/24"]
  network_id     = "${yandex_vpc_network.network-1.id}"
}

resource "yandex_compute_placement_group" "group1" {
  name = "test-pg1"
}

#resource "yandex_compute_placement_group" "group2" {
#  name = "test-pg2"
#}

resource "yandex_compute_snapshot" "snapshot-1" {
  name           = "disk-snapshot"
  source_disk_id = "${yandex_compute_instance.vm[0].boot_disk[0].disk_id}"
}
```
![image](https://github.com/user-attachments/assets/8e811415-e02e-45df-bf63-145389560e9f)
![image](https://github.com/user-attachments/assets/2f3b48f4-5c6c-450a-a4dc-25de6cde7358)
![image](https://github.com/user-attachments/assets/68609d7f-0961-4266-ae4c-2ff34bf60544)


