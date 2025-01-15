# Домашнее задание к занятию «Организация сети»

### Подготовка к выполнению задания

1. Домашнее задание состоит из обязательной части, которую нужно выполнить на провайдере Yandex Cloud, и дополнительной части в AWS (выполняется по желанию). 
2. Все домашние задания в блоке 15 связаны друг с другом и в конце представляют пример законченной инфраструктуры.  
3. Все задания нужно выполнить с помощью Terraform. Результатом выполненного домашнего задания будет код в репозитории. 
4. Перед началом работы настройте доступ к облачным ресурсам из Terraform, используя материалы прошлых лекций и домашнее задание по теме «Облачные провайдеры и синтаксис Terraform». Заранее выберите регион (в случае AWS) и зону.

---
### Задание 1. Yandex Cloud 

**Что нужно сделать**

1. Создать пустую VPC. Выбрать зону.
2. Публичная подсеть.

 - Создать в VPC subnet с названием public, сетью 192.168.10.0/24.
 - Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1.
 - Создать в этой публичной подсети виртуалку с публичным IP, подключиться к ней и убедиться, что есть доступ к интернету.
3. Приватная подсеть.
 - Создать в VPC subnet с названием private, сетью 192.168.20.0/24.
 - Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс.
 - Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее, и убедиться, что есть доступ к интернету.

Resource Terraform для Yandex Cloud:

- [VPC subnet](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_subnet).
- [Route table](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/vpc_route_table).
- [Compute Instance](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/compute_instance).

### Выполнение задания 1. Yandex Cloud

1. Создаю пустую VPC с именем dvl:

```
resource "yandex_vpc_network" "dvl" {
  name = var.vpc_name

variable "vpc_name" {
  type        = string
  default     = "dvl"
  description = "VPC network"
}
```

2. Создаю в VPC публичную подсеть с названием public, сетью 192.168.10.0/24:

```
resource "yandex_vpc_subnet" "public" {
  name           = var.public_subnet
  zone           = var.default_zone
  network_id     = yandex_vpc_network.dvl.id
  v4_cidr_blocks = var.public_cidr
}

variable "public_cidr" {
  type        = list(string)
  default     = ["192.168.10.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "public_subnet" {
  type        = string
  default     = "public"
  description = "subnet name"
}
```

[network.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/network.tf)

[variables.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/variables.tf)

3. Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1.

<details><summary>Листинг nat.tf.</summary>
 
```
variable "yandex_compute_instance_nat" {
  type        = list(object({
    vm_name = string
    cores = number
    memory = number
    core_fraction = number
    hostname = string
    platform_id = string
  }))

  default = [{
      vm_name = "nat"
      cores         = 2
      memory        = 2
      core_fraction = 5
      hostname = "nat"
      platform_id = "standard-v1"
    }]
}

variable "boot_disk_nat" {
  type        = list(object({
    size = number
    type = string
    image_id = string
    }))
    default = [ {
    size = 10
    type = "network-hdd"
    image_id = "fd80mrhj8fl2oe87o4e1"
  }]
}

resource "yandex_compute_instance" "nat" {
  name        = var.yandex_compute_instance_nat[0].vm_name
  platform_id = var.yandex_compute_instance_nat[0].platform_id
  hostname = var.yandex_compute_instance_nat[0].hostname

  resources {
    cores         = var.yandex_compute_instance_nat[0].cores
    memory        = var.yandex_compute_instance_nat[0].memory
    core_fraction = var.yandex_compute_instance_nat[0].core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = var.boot_disk_nat[0].image_id
      type     = var.boot_disk_nat[0].type
      size     = var.boot_disk_nat[0].size
    }
  }

  metadata = {
    ssh-keys = "ubuntu:${local.ssh-keys}"
    serial-port-enable = "1"
  }

  network_interface {
    subnet_id  = yandex_vpc_subnet.public.id
    nat        = true
    ip_address = "192.168.10.254"
  }
  scheduling_policy {
    preemptible = true
  }
}
```

</details>

[nat.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/nat.tf)

4. Создаю в публичной подсети виртуальную машину с публичным IP.

<details><summary>Листинг public.tf</summary>

```
variable "yandex_compute_instance_public" {
  type        = list(object({
    vm_name = string
    cores = number
    memory = number
    core_fraction = number
    hostname = string
    platform_id = string
  }))

  default = [{
      vm_name = "public"
      cores         = 2
      memory        = 2
      core_fraction = 5
      hostname = "public"
      platform_id = "standard-v1"
    }]
}

variable "boot_disk_public" {
  type        = list(object({
    size = number
    type = string
    image_id = string
    }))
    default = [ {
    size = 10
    type = "network-hdd"
    image_id = "fd8pbf0hl06ks8s3scqk"
  }]
}

resource "yandex_compute_instance" "public" {
  name        = var.yandex_compute_instance_public[0].vm_name
  platform_id = var.yandex_compute_instance_public[0].platform_id
  hostname = var.yandex_compute_instance_public[0].hostname

  resources {
    cores         = var.yandex_compute_instance_public[0].cores
    memory        = var.yandex_compute_instance_public[0].memory
    core_fraction = var.yandex_compute_instance_public[0].core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = var.boot_disk_public[0].image_id
      type     = var.boot_disk_public[0].type
      size     = var.boot_disk_public[0].size
    }
  }

  metadata = {
    ssh-keys = "ubuntu:${local.ssh-keys}"
    serial-port-enable = "1"
  }

  network_interface {
    subnet_id  = yandex_vpc_subnet.public.id
    nat        = true
  }
  scheduling_policy {
    preemptible = true
  }
}
```

</details>

[public.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/public.tf)

Проверка доступа к интернету с виртуальной машины:

```
[root@localhost hw1]# ssh ubuntu@89.169.139.196
.
.
.
ubuntu@public:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=58 time=24.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=58 time=21.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=58 time=63.4 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=58 time=46.5 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3068ms
rtt min/avg/max/mdev = 21.727/39.131/63.424/16.951 ms
```

5. Создать в VPC subnet с названием private, сетью 192.168.20.0/24.

```
resource "yandex_vpc_subnet" "private" {
  name           = var.private_subnet
  zone           = var.default_zone
  network_id     = yandex_vpc_network.dvl.id
  v4_cidr_blocks = var.private_cidr
  route_table_id = yandex_vpc_route_table.private-route.id
}

variable "private_cidr" {
  type        = list(string)
  default     = ["192.168.20.0/24"]
  description = "https://cloud.yandex.ru/docs/vpc/operations/subnet-create"
}

variable "private_subnet" {
  type        = string
  default     = "private"
  description = "subnet name"
}
```

[network.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/network.tf)

[variables.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/variables.tf)

6. Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс.

```
resource "yandex_vpc_route_table" "private-route" {
  name       = "private-route"
  network_id = yandex_vpc_network.dvl.id
  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = "192.168.10.254"
  }
}
```

7. Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее, и убедиться, что есть доступ к интернету.

<details><summary>Листинг private.tf</summary>

```
variable "yandex_compute_instance_private" {
  type        = list(object({
    vm_name = string
    cores = number
    memory = number
    core_fraction = number
    hostname = string
    platform_id = string
  }))

  default = [{
      vm_name = "private"
      cores         = 2
      memory        = 2
      core_fraction = 5
      hostname = "private"
      platform_id = "standard-v1"
    }]
}

variable "boot_disk_private" {
  type        = list(object({
    size = number
    type = string
    image_id = string
    }))
    default = [ {
    size = 10
    type = "network-hdd"
    image_id = "fd8pbf0hl06ks8s3scqk"
  }]
}

resource "yandex_compute_instance" "private" {
  name        = var.yandex_compute_instance_private[0].vm_name
  platform_id = var.yandex_compute_instance_private[0].platform_id
  hostname = var.yandex_compute_instance_private[0].hostname

  resources {
    cores         = var.yandex_compute_instance_private[0].cores
    memory        = var.yandex_compute_instance_private[0].memory
    core_fraction = var.yandex_compute_instance_private[0].core_fraction
  }

  boot_disk {
    initialize_params {
      image_id = var.boot_disk_private[0].image_id
      type     = var.boot_disk_private[0].type
      size     = var.boot_disk_private[0].size
    }
  }

  metadata = {
    ssh-keys = "ubuntu:${local.ssh-keys}"
    serial-port-enable = "1"
  }

  network_interface {
    subnet_id  = yandex_vpc_subnet.private.id
    nat        = false
  }
  scheduling_policy {
    preemptible = true
  }
}
```

<details>

[private.tf](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/private.tf)

Проверка доступности интернета на приватной виртуальной машине и работы NAT-инстанса:

```
[root@localhost hw1]# scp ~/.ssh/id_ed25519 ubuntu@89.169.139.196:/home/ubuntu/.ssh
id_ed25519                                                                                                 100%  419    26.0KB/s   00:00
[root@localhost hw1]# ssh ubuntu@89.169.139.196
.
.
.
ubuntu@public:~$ ssh ubuntu@192.168.20.18
.
.
.
ubuntu@private:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=54 time=21.0 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=54 time=19.7 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=54 time=19.7 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=54 time=28.4 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 19.676/22.210/28.432/3.632 ms
ubuntu@private:~$
```

Отключим виртуальную машину с NAT-инстансом и снова проверим доступ в интернет:

![изображение](https://github.com/stepynin-georgy/hw_cloud_1/blob/main/img/Screenshot_3.png)

```
ubuntu@private:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3055ms

ubuntu@private:~$
```

Интернет на приватной виртуальной машине перестал работать после отключения NAT-инстанса, следовательно статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс был настроен корректно.

Output вывод Terraform выглядит следующим образом:

```
[root@localhost hw1]# terraform output
nat_instance_info = {
  "ip_external" = "84.201.172.242"
  "ip_internal" = "192.168.10.254"
  "name" = "nat"
  "network" = "dvl"
  "subnet" = "public"
}
private_vm_info = {
  "ip_external" = ""
  "ip_internal" = "192.168.20.18"
  "name" = "private"
  "network" = "dvl"
  "subnet" = "private"
}
public_vm_info = {
  "ip_external" = "89.169.139.196"
  "ip_internal" = "192.168.10.3"
  "name" = "public"
  "network" = "dvl"
  "subnet" = "public"
}
```

---
### Задание 2. AWS* (задание со звёздочкой)

Это необязательное задание. Его выполнение не влияет на получение зачёта по домашней работе.

**Что нужно сделать**

1. Создать пустую VPC с подсетью 10.10.0.0/16.
2. Публичная подсеть.

 - Создать в VPC subnet с названием public, сетью 10.10.1.0/24.
 - Разрешить в этой subnet присвоение public IP по-умолчанию.
 - Создать Internet gateway.
 - Добавить в таблицу маршрутизации маршрут, направляющий весь исходящий трафик в Internet gateway.
 - Создать security group с разрешающими правилами на SSH и ICMP. Привязать эту security group на все, создаваемые в этом ДЗ, виртуалки.
 - Создать в этой подсети виртуалку и убедиться, что инстанс имеет публичный IP. Подключиться к ней, убедиться, что есть доступ к интернету.
 - Добавить NAT gateway в public subnet.
3. Приватная подсеть.
 - Создать в VPC subnet с названием private, сетью 10.10.2.0/24.
 - Создать отдельную таблицу маршрутизации и привязать её к private подсети.
 - Добавить Route, направляющий весь исходящий трафик private сети в NAT.
 - Создать виртуалку в приватной сети.
 - Подключиться к ней по SSH по приватному IP через виртуалку, созданную ранее в публичной подсети, и убедиться, что с виртуалки есть выход в интернет.

Resource Terraform:

1. [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc).
1. [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet).
1. [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway).

### Правила приёма работы

Домашняя работа оформляется в своём Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
