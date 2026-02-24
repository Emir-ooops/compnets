# ЛР 3. Ansible + Caddy

## Задача

Развернуть веб-сервер Caddy при помощи Ansible Playbook, настроить конфигурацию через шаблон Jinja2, реализовать логирование и развернуть собственную веб-страницу. Работа выполнена в среде Ubuntu (WSL) на Windows.

---

## Ход работы

### 1. Установка Ansible

Обновление системы:

```bash
sudo apt update
sudo apt upgrade -y
```

Установка Ansible:

```bash
sudo apt install ansible -y
```

Проверка версии:

```bash
ansible --version
```

![1](https://github.com/user-attachments/assets/0d976e04-3a58-4bef-b6a3-11ffb3a2a494)


---

### 2. Создание структуры проекта

Создание рабочей директории:

```bash
mkdir lab3
cd lab3
```

Создание структуры проекта:

```bash
mkdir inventory
mkdir roles
```

Создание конфигурационного файла `ansible.cfg`:

```ini
[defaults]
host_key_checking = false
inventory = inventory/hosts
```

Создание файла `inventory/hosts`:

```ini
[my_servers]
local_server ansible_host=localhost
```

Проверка подключения:

```bash
ansible my_servers -m ping -c local
```

![2](https://github.com/user-attachments/assets/46f35b2e-2282-40c2-96a3-28bccacee260)

---

### 3. Создание роли

```bash
cd roles
ansible-galaxy init caddy_deploy
cd ..
```

![3](https://github.com/user-attachments/assets/f51911d0-5881-436e-a8af-dc8aa1823736)


---

### 4. Настройка tasks

Файл `roles/caddy_deploy/tasks/main.yml`:

```yaml
---
- name: Install prerequisites
  apt:
    pkg:
      - debian-keyring
      - debian-archive-keyring
      - apt-transport-https
      - curl
    update_cache: yes
  become: yes

- name: Add Caddy GPG key
  apt_key:
    url: https://dl.cloudsmith.io/public/caddy/stable/gpg.key
    state: present
    keyring: /usr/share/keyrings/caddy-stable-archive-keyring.gpg
  become: yes

- name: Add Caddy repo
  apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable
  become: yes

- name: Install Caddy
  apt:
    name: caddy
    state: present
    update_cache: yes
  become: yes

- name: Ensure log file exists and has correct permissions
  file:
    path: /var/log/caddy_access.log
    state: touch
    owner: caddy
    group: caddy
    mode: '0644'
  become: yes

- name: Create custom index.html
  copy:
    content: |
      <html>
      <body>
        <h1>Эмир делает 3 лабу по компсету</h1>
      </body>
      </html>
    dest: /usr/share/caddy/index.html
  become: yes

- name: Deploy Caddyfile from template
  template:
    src: templates/Caddyfile.j2
    dest: /etc/caddy/Caddyfile
  become: yes

- name: Reload Caddy
  service:
    name: caddy
    state: reloaded
  become: yes
```

---

### 5. Настройка шаблона

Файл `roles/caddy_deploy/templates/Caddyfile.j2`:

```jinja
:80 {
  root * /usr/share/caddy
  file_server

  log {
    output file {{ log.file }}
    format json
    level {{ log.level }}
  }
}
```

---

### 6. Настройка переменных

Файл `roles/caddy_deploy/vars/main.yml`:

```yaml
---
log:
  file: /var/log/caddy_access.log
  level: INFO
```

---

### 7. Создание Playbook

Файл `caddy_deploy.yml`:

```yaml
---
- name: Install and configure Caddy
  hosts: my_servers
  connection: local
  roles:
    - caddy_deploy
```

---

### 8. Запуск Playbook

```bash
sudo ansible-playbook caddy_deploy.yml
```

После выполнения отображается `PLAY RECAP` с `failed=0`.

![4](https://github.com/user-attachments/assets/8345babf-b708-4d98-9592-95bea7da6917)


---

### 9. Проверка работы веб-сервера

Открываем в браузере:

```
http://localhost
```

Отображается собственная страница:

**Hello from Ansible Lab 3**

![6](https://github.com/user-attachments/assets/ef00e4fa-6201-4c26-88c9-326c20a99045)


---

### 10. Проверка логирования

```bash
sudo tail -n 20 /var/log/caddy_access.log
```

В файле отображаются JSON-записи HTTP-запросов.

![8](https://github.com/user-attachments/assets/3e11e51f-4e40-4e5f-be87-7dc6b87006ae)


---

## Вывод

В ходе лабораторной работы был установлен и настроен Ansible, создана роль для автоматической установки веб-сервера Caddy, использован шаблон Jinja2 для генерации конфигурационного файла, настроено логирование и развернута собственная веб-страница. Playbook выполняется без ошибок, веб-сервер функционирует корректно.
