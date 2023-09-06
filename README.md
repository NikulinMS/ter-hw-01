# Домашнее задание к занятию "`Введение в Terraform`" - `Никулин Михаил Сергеевич`



---

### Задание 1

Содержимое файла .gitignore:

```
# Local .terraform directories and files
**/.terraform/*
.terraform*

# .tfstate files
*.tfstate
*.tfstate.*

# own secret vars store.
personal.auto.tfvars
```
Согласно файлу личную информацию можно сохранить в файл в каталог `.terraform/` Однако это не касается подкаталогов. Также можно использовать файл `personal.auto.tfvars`

Содержимое state-файла после исполнения:
```
{
  "version": 4,
  "terraform_version": "1.4.6",
  "serial": 5,
  "lineage": "72c99934-3bab-83d3-0736-2f735d1ed099",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "random_password",
      "name": "random_string",
      "provider": "provider[\"registry.terraform.io/hashicorp/random\"]",
      "instances": [
        {
          "schema_version": 3,
          "attributes": {
            "bcrypt_hash": "$2a$10$SwDc7O5yLLOorzVaMLp4puT2bMxcDURhaqbYKiCgIPLehSt7aJFQC",
            "id": "none",
            "keepers": null,
            "length": 16,
            "lower": true,
            "min_lower": 1,
            "min_numeric": 1,
            "min_special": 0,
            "min_upper": 1,
            "number": true,
            "numeric": true,
            "override_special": null,
            "result": "gl25JHAjHwU69Auh",
            "special": false,
            "upper": true
          },
          "sensitive_attributes": []
        }
      ]
    }
  ],
  "check_results": null

```
Нас интересует `"result": "gl25JHAjHwU69Auh"`

Результат выполнения команды `terraform validate` после раскомментирования:
```
╷
│ Error: Missing name for resource
│
│   on main.tf line 24, in resource "docker_image":
│   24: resource "docker_image" {
│
│ All resource blocks must have 2 labels (type, name).
╵
```
Ошибка означает, что не задано имя ресурса, указан только тип
```
╷
│ Error: Invalid resource name
│
│   on main.tf line 29, in resource "docker_container" "1nginx":
│   29: resource "docker_container" "1nginx" {
│
│ A name must start with a letter or underscore and may contain only letters, digits, underscores,
│ and dashes.
╵
```
Ошибка означает, что имя ресурса должно всегда начинаться с буквы или символа подчеркивания "_"

После решения этих ошибок и повторного запуска команды `terraform validate` получаем новую ошибку:
```
╷
│ Error: Reference to undeclared resource
│
│   on main.tf line 31, in resource "docker_container" "nginx":
│   31:   name  = "example_${random_password.random_string_FAKE.resulT}"
│
│ A managed resource "random_password" "random_string_FAKE" has not been declared in the root module.
╵
```
Ошибка означает, что ресурс с типом `random_password` и именем `random_string_FAKE` не был объявлен ранее, был объявлен ресурс с именем `random_string`. Исправим ее. Сразу можно исправить ошибку обращения к аргументу `resulT`, правильно будет `result`

После решения ошибок запускаем команду `terraform plan` и `terraform apply`. Резукдьтат команды `docker ps`:
```
[root@node01 src]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                  NAMES
de678b970b6e   eea7b3dcba7e   "/docker-entrypoint.…"   33 seconds ago   Up 24 seconds   0.0.0.0:8000->80/tcp   example_gl25JHAjHwU69Auh
```
Меняем имя контейнера на `hello_world` и запускаем команду `terraform apply -auto-approve`. Результат выполнения команды `docker ps`:
```
[root@node01 src]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                  NAMES
f649dde8e055   eea7b3dcba7e   "/docker-entrypoint.…"   17 seconds ago   Up 16 seconds   0.0.0.0:8000->80/tcp   hello_world
```
Как мы видим, при выполнении команды `terraform apply -auto-approve` происходит автоматическое подтверждение выполнение запланированных операций без возможности предварительно ознакомиться с выполняемыми процедурами, только ознакомиться с результатами. В нашем случае произошла замена предыдущего контейнера на новый с новым именем, старый был утрачен.

Уничтожим все созданные объекты командой `terraform destroy`. Содержимое файла terraform.tfstate выглядит следующим образом:
```
{
  "version": 4,
  "terraform_version": "1.4.6",
  "serial": 15,
  "lineage": "72c99934-3bab-83d3-0736-2f735d1ed099",
  "outputs": {},
  "resources": [],
  "check_results": null
}
```
После удаления всех созданных объектов образ остался:
```
[root@node01 src]# docker image ls
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    eea7b3dcba7e   3 weeks ago   187MB
```
У ресурса `docker_image` есть атрибут `keep_locally`, который определяет, будет ли удален образ при удалении ресурса или нет. В нашем случае этому атрибуту передано значение `true`, что означает, что образ остается локально при удалении ресурса.

---

### Задание 2*

Подготовим `main.tf` файл:
```
terraform {
  required_providers {
    virtualbox = {
      source = "shekeriev/virtualbox"
      version = "0.0.4"
    }
  }
}

provider "virtualbox" {
  delay      = 60
  mintimeout = 5
}

resource "virtualbox_vm" "vm1" {
  name   = "centos-7"
  image  = "https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box"
  cpus      = 1
  memory    = "512 mib"


  network_adapter {
    type           = "hostonly"
    device         = "IntelPro1000MTDesktop"
    host_interface = "VirtualBox Host-Only Ethernet Adapter"
  }
}

output "IPAddress" {
  value = element(virtualbox_vm.vm1.*.network_adapter.0.ipv4_address, 1)
}
```
Результат команды `terraform plan`:
```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # virtualbox_vm.vm1 will be created
  + resource "virtualbox_vm" "vm1" {
      + cpus   = 1
      + id     = (known after apply)
      + image  = "https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box"
      + memory = "512 mib"
      + name   = "centos-7"
      + status = "running"

      + network_adapter {
          + device                 = "IntelPro1000MTDesktop"
          + host_interface         = "VirtualBox Host-Only Ethernet Adapter"
          + ipv4_address           = (known after apply)
          + ipv4_address_available = (known after apply)
          + mac_address            = (known after apply)
          + status                 = (known after apply)
          + type                   = "hostonly"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + IPAddress = (known after apply)
```
Результат работы:
```
virtualbox_vm.vm1: Creation complete after 1m44s [id=2a9bb978-b8c1-416d-bdf2-1bc054392945]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

IPAddress = "192.168.56.101"
```
![task_2_1.png](img%2Ftask_2_1.png)