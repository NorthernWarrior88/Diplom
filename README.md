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
![](https://github.com/tverdyakov/diplom_tverdyakov-sys-20/blob/main/01_Установка%20и%20подготовка%20Terraform%20и%20Ansible/screenshots/06.png)
#### Terraform готов к использованию. Продолжение в основной части.

---
### 1.2 Установка и подготовка Ansible.
Устанавливаю Ansible и проверяю версию.
```bash
apt install ansible
ansible --version
```
![](https://github.com/tverdyakov/diplom_tverdyakov-sys-20/blob/main/01_Установка%20и%20подготовка%20Terraform%20и%20Ansible/screenshots/07.png)
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
