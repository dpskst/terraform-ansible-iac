# Project 04. Terraform + Ansible Infrastructure Automation

## 1. Project Overview

Terraform과 Ansible을 활용하여 Linux 서버의 Infrastructure와 Configuration을 코드로 관리하고 자동화하는 프로젝트입니다.

Terraform을 통해 Infrastructure를 코드로 정의하고, Ansible을 통해 Linux 서버의 패키지 및 서비스 설정을 자동화했습니다.

본 프로젝트에서는 Infrastructure Provisioning과 Configuration Management를 분리하여 각각 Terraform과 Ansible의 역할을 구성했습니다.

### 주요 목표

* Terraform을 이용한 Infrastructure as Code 구성
* Libvirt 기반 Infrastructure 관리
* Terraform을 이용한 VM Resource 관리
* Ansible을 이용한 Linux 서버 Configuration Management
* SSH 기반 원격 서버 구성 자동화
* Package 및 Service 자동화
* Ansible Playbook의 Idempotency 확인
* Terraform 및 Libvirt 환경에서 발생한 문제 분석

---

## 2. Architecture

```text
                        GitHub
                          |
                          |
              +-----------+-----------+
              |                       |
              v                       v
          Terraform                Ansible
              |                       |
              |                       |
      Infrastructure Layer    Configuration Layer
              |                       |
              v                       v
           Libvirt                 SSH
              |                       |
              v                       v
          Linux VM <------------- Ansible
              |
              +----------------------+
              |
              +-- Hostname
              +-- Package
              +-- Nginx
              +-- Service
```

### Terraform

Infrastructure 영역을 담당합니다.

* VM
* CPU
* Memory
* Disk
* Network
* Libvirt Domain
* Libvirt Volume

### Ansible

Server Configuration 영역을 담당합니다.

* Hostname
* Package Installation
* Nginx Installation
* Service Configuration
* Service Enable / Start

---

## 3. Environment

### Host

| Item           | Configuration         |
| -------------- | --------------------- |
| OS             | Rocky Linux 8.10      |
| CPU            | Intel Xeon E5-2620 v2 |
| Memory         | 62 GB                 |
| Virtualization | KVM                   |
| Libvirt        | 8.0.0                 |
| QEMU           | 6.2.0                 |

### VM

| Item    | Configuration           |
| ------- | ----------------------- |
| OS      | Rocky Linux 9.8 Minimal |
| VM      | web01                   |
| vCPU    | 2                       |
| Memory  | 2 GB                    |
| Disk    | 15 GB                   |
| Network | Libvirt Default Network |
| IP      | 192.168.122.88          |
| Service | Nginx                   |

### Software

| Software         | Version   |
| ---------------- | --------- |
| Terraform        | 1.16.4    |
| Libvirt Provider | 0.9.9     |
| Ansible          | Installed |
| KVM              | Enabled   |
| Libvirt          | 8.0.0     |
| QEMU             | 6.2.0     |

---

## 4. Terraform

### 4.1 Provider Configuration

Terraform과 Libvirt 환경을 연동하기 위해 Libvirt Provider를 구성했습니다.

```hcl
terraform {
  required_providers {
    libvirt = {
      source  = "dmacvicar/libvirt"
      version = "~> 0.9"
    }
  }
}

provider "libvirt" {
  uri = "qemu:///system"
}
```

Terraform 초기화:

```bash
terraform init
```

Provider가 정상적으로 초기화되는 것을 확인했습니다.

### 4.2 Terraform Resource

Libvirt Volume을 Terraform Resource로 정의했습니다.

```hcl
resource "libvirt_volume" "web02_disk" {
  name     = "web02.qcow2"
  pool     = "project04"
  capacity = 15 * 1024 * 1024 * 1024

  target = {
    format = {
      type = "qcow2"
    }
  }
}
```

Libvirt Domain 역시 Terraform으로 정의했습니다.

```hcl
resource "libvirt_domain" "web02" {
  name = "web02"
  type = "kvm"

  memory = 2048
  vcpu   = 2

  running   = true
  autostart = false
}
```

### 4.3 Terraform Workflow

```text
terraform init
        |
        v
terraform plan
        |
        v
terraform apply
        |
        v
Infrastructure
        |
        v
terraform state
        |
        v
terraform destroy
```

### 4.4 Terraform Plan

```bash
terraform plan
```

Terraform Configuration과 실제 Infrastructure 상태를 비교하여
변경 사항을 확인했습니다.

<img width="1031" height="469" alt="image" src="https://github.com/user-attachments/assets/96fe7066-cf1a-47bc-a2c7-d9f483c5fc6a" />

현재 Terraform Configuration과 실제 Infrastructure 상태가 일치하여
추가적인 변경 사항이 없는 것을 확인했습니다.

### 4.5 Terraform Apply

```bash
terraform apply
```

Terraform을 이용하여 현재 Infrastructure 상태를 확인하고,
Terraform Configuration과 실제 Infrastructure 상태가 일치하는 것을 확인했습니다.


현재 관리 중인 Infrastructure에 추가적인 변경 사항이 없어
Resource가 그대로 유지되고 있음을 확인했습니다.

<img width="895" height="162" alt="image" src="https://github.com/user-attachments/assets/f7831940-6825-4089-9156-f3c9d7c07c93" />


### 4.6 Terraform State

Terraform State를 이용하여 관리되고 있는 Resource를 확인했습니다.

```bash
terraform state list
```

예:

```text
libvirt_volume.web02_disk
libvirt_domain.web02
```

---

## 5. Ansible

### 5.1 Inventory

Ansible을 이용하여 Linux 서버를 관리하기 위해 Inventory를 구성했습니다.

```ini
[web]
web01 ansible_host=192.168.122.88

[web:vars]
ansible_user=devops
```

### 5.2 Playbook

Linux 서버의 기본적인 Configuration을 자동화했습니다.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  tasks:
    - name: Set hostname
      ansible.builtin.hostname:
        name: web01

    - name: Install required packages
      ansible.builtin.dnf:
        name:
          - vim
          - curl
          - wget
          - git
          - unzip
        state: present

    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present

    - name: Enable and start Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: true
```

### 5.3 Playbook Execution

```bash
ansible-playbook \
  -i ansible/inventory/hosts.ini \
  ansible/playbooks/site.yml
```

실행 결과:

```text
ok=5
changed=4
unreachable=0
failed=0
skipped=0
rescued=0
ignored=0
```

<img width="812" height="366" alt="image" src="https://github.com/user-attachments/assets/e26a4234-fc02-4142-a7f0-2a794c37dbcd" />


---

## 6. Linux Server Configuration

Ansible을 통해 Linux 서버의 Configuration을 자동화했습니다.

### Hostname

```text
web01
```

### Package

```text
vim
curl
wget
git
unzip
nginx
```

### Service

```text
nginx
```

Nginx를 설치하고 Systemd를 이용하여 서비스를 활성화했습니다.

```bash
systemctl enable nginx
systemctl start nginx
```

---

## 7. Ansible Idempotency

Ansible Playbook을 반복 실행하여 Idempotency를 확인했습니다.

동일한 Playbook을 여러 번 실행해도 이미 적용된 Configuration은 불필요하게 변경되지 않도록 구성했습니다.

```bash
ansible-playbook \
  -i ansible/inventory/hosts.ini \
  ansible/playbooks/site.yml
```

첫 번째 실행:

```text
changed > 0
```

Configuration이 적용된 이후 동일한 Playbook을 다시 실행하면 변경사항이 최소화됩니다.

```text
changed=0
```

이를 통해 Ansible의 Idempotent Configuration Management 방식을 확인했습니다.

---

## 8. Nginx Configuration

Ansible을 통해 Nginx를 자동으로 설치하고 서비스를 활성화했습니다.

서비스 상태 확인:

```bash
systemctl status nginx --no-pager
```

또는 Ansible을 이용하여 원격 서버에서 확인:

```bash
ansible web \
  -i ansible/inventory/hosts.ini \
  -m shell \
  -a "systemctl status nginx --no-pager"
```

<img width="721" height="350" alt="image" src="https://github.com/user-attachments/assets/46304f15-c7c2-48ef-8f9a-46a723624dd4" />


---

## 9. Terraform + Ansible Workflow

본 프로젝트에서는 Terraform과 Ansible의 역할을 분리했습니다.

```text
                    Terraform
                       |
                       v
              Infrastructure
                       |
                       v
                   Linux VM
                       |
                       |
                       v
                    Ansible
                       |
                       v
             Server Configuration
                       |
              +--------+--------+
              |        |        |
              v        v        v
          Package   Nginx    Service
```

Terraform:

```text
Infrastructure Provisioning
```

Ansible:

```text
Configuration Management
```

각 도구의 역할을 분리하여 Infrastructure와 Server Configuration을 관리했습니다.

---

## 10. Troubleshooting

### 10.1 Libvirt VM Boot Issue

Terraform을 이용하여 `web02` VM을 생성하는 과정에서 VM이 정상적으로 부팅되지 않고 다음 상태가 발생했습니다.

```text
web02 paused
```

QEMU Log를 확인했습니다.

```bash
tail -50 /var/log/libvirt/qemu/web02.log
```

확인된 오류:

```text
KVM internal error. Suberror: 3
```

### 10.2 Domain XML 확인

Terraform Configuration과 실제 Libvirt Domain XML을 비교했습니다.

Machine Type:

```xml
<type arch='x86_64' machine='pc-q35-rhel8.6.0'>hvm</type>
```

CPU:

```xml
<cpu mode='custom' match='exact' check='full'>
  <model fallback='forbid'>IvyBridge-IBRS</model>
</cpu>
```

ACPI / APIC:

```xml
<features>
  <acpi/>
  <apic/>
</features>
```

### 10.3 QEMU Process 확인

실제 QEMU Process를 확인하여 Terraform Configuration이 QEMU 실행 옵션에 반영되는지 확인했습니다.

```bash
ps -ef | grep '[q]emu.*web02'
```

확인:

```text
-machine pc-q35-rhel8.6.0
-accel kvm
-cpu IvyBridge-IBRS
```

정상 실행 중인 `web01`과 QEMU 실행 옵션 및 Domain XML을 비교하여 문제를 단계적으로 분석했습니다.

<img width="615" height="217" alt="image" src="https://github.com/user-attachments/assets/01f026c9-2222-48a2-b2fa-12d8372c79bb" />


해당 문제는 Terraform Resource 생성 자체의 오류가 아닌 KVM/QEMU 실행 단계에서 발생하는 문제로 확인했으며, 본 프로젝트에서는 Troubleshooting 과정으로 기록했습니다.

---

## 11. Project Structure

```text
project-04-iac/
│
├── README.md
│
├── terraform/
│   ├── main.tf
│   └── terraform.tfstate
│
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini
│   │
│   └── playbooks/
│       └── site.yml
│
└── docs/
    ├── images/
    │   ├── project04-architecture.png
    │   ├── terraform-plan.png
    │   ├── terraform-apply.png
    │   ├── ansible-result.png
    │   ├── nginx-status.png
    │   └── troubleshooting.png
    │
    └── installation.md
```

---

## 12. GitHub Repository

프로젝트 전체 소스와 Configuration 파일은 GitHub Repository에서 관리합니다.

```text
Terraform
 ├── main.tf
 └── terraform.tfstate

Ansible
 ├── inventory
 └── playbooks

Documentation
 ├── README.md
 └── installation.md
```

---

## 13. Project Goals

* [x] Terraform Provider 구성
* [x] Terraform Init / Plan / Apply / Destroy
* [x] Libvirt Volume 관리
* [x] Libvirt Domain 정의
* [x] VM CPU / Memory / Disk / Network 코드화
* [x] Ansible Inventory 구성
* [x] SSH 기반 원격 Configuration
* [x] Linux Package 자동 설치
* [x] Nginx 자동 설치
* [x] Systemd Service 관리
* [x] Ansible Idempotency 확인
* [x] Terraform / Libvirt Troubleshooting
* [ ] Ansible Role 기반 Configuration
* [ ] Linux Security Configuration Automation

---

## 14. What I Learned

### Terraform

Terraform을 이용하여 Infrastructure를 코드로 정의하고 `plan`, `apply`, `destroy`, `state`를 통해 Infrastructure Lifecycle을 관리하는 방법을 학습했습니다.

### Ansible

Ansible을 이용하여 SSH 기반으로 Linux 서버에 접속하고 Package, Hostname, Service 등의 Configuration을 자동화하는 방법을 학습했습니다.

### Infrastructure as Code

Terraform과 Ansible을 사용하면서 Infrastructure Provisioning과 Configuration Management를 분리하여 관리하는 구조를 이해했습니다.

```text
Terraform
    |
    v
Infrastructure
    |
    v
Linux Server
    |
    v
Ansible
    |
    v
Configuration
```

### Troubleshooting

Terraform Configuration뿐만 아니라 실제 Libvirt XML과 QEMU Process를 확인하여 Infrastructure 문제를 계층별로 분석하는 과정을 경험했습니다.
