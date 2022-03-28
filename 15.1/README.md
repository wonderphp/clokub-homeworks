# Домашнее задание к занятию "15.1. Организация сети"

Домашнее задание будет состоять из обязательной части, которую необходимо выполнить на провайдере Яндекс.Облако и дополнительной части в AWS по желанию. Все домашние задания в 15 блоке связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
Все задания требуется выполнить с помощью Terraform, результатом выполненного домашнего задания будет код в репозитории. 

Перед началом работ следует настроить доступ до облачных ресурсов из Terraform используя материалы прошлых лекций и [ДЗ](https://github.com/netology-code/virt-homeworks/tree/master/07-terraform-02-syntax ). А также заранее выбрать регион (в случае AWS) и зону.

---
## Задание 1. Яндекс.Облако (обязательное к выполнению)

1. Создать VPC.
- Создать пустую VPC. Выбрать зону.
2. Публичная подсеть.
- Создать в vpc subnet с названием public, сетью 192.168.10.0/24.
- Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1
- Создать в этой публичной подсети виртуалку с публичным IP и подключиться к ней, убедиться что есть доступ к интернету.  

3. Приватная подсеть.
- Создать в vpc subnet с названием private, сетью 192.168.20.0/24.
- Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс
- Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее и убедиться что есть доступ к интернету  
![image](https://user-images.githubusercontent.com/30965391/160444132-709a7309-ddcc-45d4-88d9-2239bdd5216f.png)


Занищебродил рам, что бы если что, не быстро рублевые ресурсы кончались)  
![image](https://user-images.githubusercontent.com/30965391/160442928-92630704-4aec-4f6c-9662-2ec42fcf63a3.png)
Нашел косяк, глупо так получилось. Может незаметил как мышой прожал.  
resource "yandex_vpc_subnet" "private" {  
  zone           = "ru-central1-b"  
  network_id     = yandex_vpc_network.test_network.id  
  v4_cidr_blocks = ["192.168.20.0/24"]  
  name               = "private"  
**route_table_id = "${yandex_vpc_route_table.test-rt-a.id}"**  
}  
  
  
resource "yandex_vpc_route_table" **"test-rt-a"** {  
  network_id = "${yandex_vpc_network.test_network.id}"  
  
  static_route {   
    destination_prefix = "0.0.0.0/0"  
    next_hop_address   = "192.168.10.254"  
 }  
}  
  


```
pixel@kopilka:~/15.1$ cat main.tf
terraform {
  required_providers {
    yandex = {
      source = "terraform-registry.storage.yandexcloud.net/yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  token     = "спрятано"
  cloud_id  = "спрятано"
  folder_id = "b1g1e417rlclna6nbfk6"
  zone      = "ru-central1-b"

}

resource "yandex_vpc_network" "test_network" {}

resource "yandex_vpc_subnet" "public" {
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.test_network.id
  v4_cidr_blocks = ["192.168.10.0/24"]
  name               = "public"
}

resource "yandex_vpc_subnet" "private" {
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.test_network.id
  v4_cidr_blocks = ["192.168.20.0/24"]
  name               = "private"
  route_table_id = "${yandex_vpc_route_table.test-rt-a.id}"
}

data "yandex_compute_image" "test_image" {
  image_id = "fd8ni66vim3em9jgua7g"
}


resource "yandex_vpc_route_table" "test-rt-a" {
  network_id = "${yandex_vpc_network.test_network.id}"

  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = "192.168.10.254"
 }
}

resource "yandex_compute_instance" "test-public" {
  name               = "test-public"
 service_account_id = "ajer57ueru2c21ng72lf"
    platform_id = "standard-v2"
    resources {
      memory = 0.5
      cores  = 2
      core_fraction = 5
    }
    boot_disk {
      mode = "READ_WRITE"
      initialize_params {
      image_id = data.yandex_compute_image.test_image.id

      }
    }
    network_interface {
      subnet_id = "${yandex_vpc_subnet.public.id}"
      nat = "true"
    }
      metadata = {
      username = "ubuntu"
      password = "ubuntu"
      ssh-keys = "ubuntu:${file("pubkey.pem.pub")}"
    }

}

resource "yandex_compute_instance" "test-private" {
  name               = "test-private"
 service_account_id = "ajer57ueru2c21ng72lf"
    platform_id = "standard-v2"
    resources {
      memory = 0.5
      cores  = 2
      core_fraction = 5
    }
    boot_disk {
      mode = "READ_WRITE"
      initialize_params {
      image_id = data.yandex_compute_image.test_image.id

      }
    }
    network_interface {
      subnet_id = "${yandex_vpc_subnet.private.id}"

    }
      metadata = {
      username = "ubuntu"
      password = "ubuntu"
      ssh-keys = "ubuntu:${file("pubkey.pem.pub")}"
    }

}

resource "yandex_compute_instance" "nat-instance" {
  name               = "nat-instance"
 service_account_id = "ajer57ueru2c21ng72lf"
    platform_id = "standard-v2"
    resources {
      memory = 0.5
      cores  = 2
      core_fraction = 5
    }
    boot_disk {
      mode = "READ_WRITE"
      initialize_params {
      image_id = "fd85vbr6kin3r8ro2e95"

      }
    }
    network_interface {
      subnet_id = "${yandex_vpc_subnet.public.id}"
      nat = "true"
      ip_address="192.168.10.254"
    }
      metadata = {
      username = "ubuntu"
      password = "ubuntu"
      ssh-keys = "ubuntu:${file("pubkey.pem.pub")}"
    }

}

```
Resource terraform для ЯО
- [VPC subnet](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_subnet)
- [Route table](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_route_table)
- [Compute Instance](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance)
---
## Задание 2*. AWS (необязательное к выполнению)

1. Создать VPC.
- Cоздать пустую VPC с подсетью 10.10.0.0/16.
2. Публичная подсеть.
- Создать в vpc subnet с названием public, сетью 10.10.1.0/24
- Разрешить в данной subnet присвоение public IP по-умолчанию. 
- Создать Internet gateway 
- Добавить в таблицу маршрутизации маршрут, направляющий весь исходящий трафик в Internet gateway.
- Создать security group с разрешающими правилами на SSH и ICMP. Привязать данную security-group на все создаваемые в данном ДЗ виртуалки
- Создать в этой подсети виртуалку и убедиться, что инстанс имеет публичный IP. Подключиться к ней, убедиться что есть доступ к интернету.
- Добавить NAT gateway в public subnet.
3. Приватная подсеть.
- Создать в vpc subnet с названием private, сетью 10.10.2.0/24
- Создать отдельную таблицу маршрутизации и привязать ее к private-подсети
- Добавить Route, направляющий весь исходящий трафик private сети в NAT.
- Создать виртуалку в приватной сети.
- Подключиться к ней по SSH по приватному IP через виртуалку, созданную ранее в публичной подсети и убедиться, что с виртуалки есть выход в интернет.

Resource terraform
- [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
- [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)
