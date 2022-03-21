# Домашнее задание к занятию "15.1. Организация сети"

Настроить Production like сеть в рамках одной зоны с помощью terraform. Модуль VPC умеет автоматически делать все что есть в этом задании. Но мы воспользуемся более низкоуровневыми абстракциями, чтобы понять, как оно устроено внутри.

1. Создать VPC.

- Используя vpc-модуль terraform, создать пустую VPC с подсетью 172.31.0.0/16.
- Выбрать регион и зону.  

```
resource "aws_vpc" "netology-hwork" {
    cidr_block           = "172.31.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support   = true
    availability_zone    = "eu-west-3a"
    instance_tenancy     = "default"

    tags {
        "Name" = "netology-hwork"
    }
}
```
2. Публичная сеть.

- Создать в vpc subnet с названием public, сетью 172.31.32.0/19 и Internet gateway.  

```
resource "aws_subnet" "subnet-02fd3c8c7f06da285-public" {
    vpc_id                  = "vpc-0b617728f4302e3cb"
    cidr_block              = "172.31.32.0/19"
    availability_zone       = "eu-west-3a"
    map_public_ip_on_launch = true

    tags {
        "Name" = "public"
    }
}

resource "aws_internet_gateway" "netology-hwork" {
    vpc_id = "vpc-0b617728f4302e3cb"

    tags {
        "Name" = "netology-hwork"
    }
}
```

- Добавить RouteTable, направляющий весь исходящий трафик в Internet gateway.  

```
resource "aws_route_table" "public" {
    vpc_id     = "vpc-0b617728f4302e3cb"

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "igw-0e371bcc7e52f75c8"
    }

    tags {
        "Name" = "public"
    }
}
```
- Создать в этой приватной сети виртуалку с публичным IP и подключиться к ней, убедиться что есть доступ к интернету.  

```
resource "aws_security_group" "vpc-0b617728f4302e3cb-default" {
    name        = "default"
    description = "default VPC security group"
    vpc_id      = "vpc-0b617728f4302e3cb"

    ingress {
        from_port       = 0
        to_port         = 0
        protocol        = "-1"
        security_groups = []
        self            = true
    }

    ingress {
        from_port       = 22
        to_port         = 22
        protocol        = "tcp"
        cidr_blocks     = ["0.0.0.0/0"]
    }

    ingress {
        from_port       = -1
        to_port         = -1
        protocol        = "icmp"
        cidr_blocks     = ["0.0.0.0/0"]
    }


    egress {
        from_port       = 0
        to_port         = 0
        protocol        = "-1"
        cidr_blocks     = ["0.0.0.0/0"]
    }

    tags {
        "Name" = "home-work"
    }
}
```
```
resource "aws_instance" "hmwork1" {
    ami                         = "ami-052f10f1c45aa2155"
    availability_zone           = "eu-west-3a"
    ebs_optimized               = false
    instance_type               = "t2.micro"
    monitoring                  = false
    key_name                    = "newkey"
    subnet_id                   = "subnet-02fd3c8c7f06da285"
    vpc_security_group_ids      = ["sg-015c82cbe199710ee"]
    associate_public_ip_address = true
    private_ip                  = "172.31.52.102"
    source_dest_check           = true

    root_block_device {
        volume_type           = "gp2"
        volume_size           = 8
        delete_on_termination = true
    }

    tags {
        "Name" = "hmwork1"
    }
}
```

![image](https://user-images.githubusercontent.com/30965391/159349744-597b2530-1da0-48bc-985b-10b59a47d3ba.png)

3. Приватная сеть.

- Создать в vpc subnet с названием private, сетью 172.31.96.0/19.  

```
resource "aws_subnet" "subnet-0b612418a7be5c797-hwork-private" {
    vpc_id                  = "vpc-0b617728f4302e3cb"
    cidr_block              = "172.31.96.0/19"
    availability_zone       = "eu-west-3a"
    map_public_ip_on_launch = false

    tags {
        "Name" = "hwork-private"
    }
}

```
- Добавить NAT gateway в public subnet.  

```
resource "aws_nat_gateway" "nat-0e577332be98a20e0" {
    allocation_id = "eipalloc-01df82154403a4049"
    subnet_id = "subnet-0b612418a7be5c797"
}
```
```
resource "aws_eip" "eipalloc-01df82154403a4049" {
    network_interface = "eni-0c190c9bf57411f20"
    vpc               = true
}
```
- Добавить Route, направляющий весь исходящий трафик private сети в NAT.  

```
resource "aws_internet_gateway" "netology-hwork" {
    vpc_id = "vpc-0b617728f4302e3cb"

    tags {
        "Name" = "netology-hwork"
    }
}
```

4. VPN.

- Настроить VPN, соединить его с сетью private.

```
resource "aws_vpn_gateway" "hwokr-vpnggw" {
    vpc_id = "vpc-0b617728f4302e3cb"
    availability_zone = ""
    tags {
        "Name" = "hwokr-vpnggw"
    }
}
```
![image](https://user-images.githubusercontent.com/30965391/159353765-269f84b3-1fdd-4ea5-b3d7-f7f06db52207.png)
![image](https://user-images.githubusercontent.com/30965391/159356133-e4d67e97-09a6-4ca7-bd86-838ed22d9813.png)

- Создать себе учетную запись и подключиться через нее.
- Создать виртуалку в приватной сети.
- Подключиться к ней по SSH по приватному IP и убедиться, что с виртуалки есть выход в интернет.
![image](https://user-images.githubusercontent.com/30965391/159368803-acbf42de-491e-40cb-9c4b-b2fd58a40677.png)
![image](https://user-images.githubusercontent.com/30965391/159369009-f7628a53-8169-4724-8e90-70fcc9747127.png)
![image](https://user-images.githubusercontent.com/30965391/159369835-21d1f355-f7f3-46c9-ae76-e5eed3c84b5a.png)


Не очень понял, что я сделал. 

p.s. Недавно поднял VPN-сервер на aws, т.к. по понятным причинам, руссофобские настроения привели к блокировке(хттп 405) ресурсов терраформ. 
(tunsafe)
Документация по AWS-ресурсам:

- [Getting started with Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-getting-started.html)

Модули terraform

- [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
- [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)
