
# Ansible Advanced Topics — Study Notes

## Topics Covered

* Ansible Roles
* Role Directory Structure
* Manual Role Creation
* Ansible Galaxy
* Creating Roles with `ansible-galaxy init`
* Nginx Role
* Ansible Vault
* Asynchronous & Polling Actions
* `import_tasks`
* `include_tasks`
* `import_playbook`
* Ansible Collections
* AWS `amazon.aws` Collection
* S3 Bucket Automation

---

# 1. Ansible Roles

Ansible Roles are a way to organize large playbooks into reusable and structured components.

Instead of keeping everything inside one large playbook, roles divide automation into separate directories containing tasks, variables, handlers, templates, and files.

### Why Use Roles?

* **Modular & Reusable** — Write once and use multiple times.
* **Scalable** — Easier to manage configurations across multiple systems.
* **Organized** — Avoid large and difficult-to-maintain playbooks.

### Simple Concept

```text
Playbook = What should happen

Role = Reusable package containing everything
       required to perform a particular job
```

---

# 2. Role Directory Structure

A typical Ansible role contains directories such as:

```text
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml
    ├── files/
    ├── handlers/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── templates/
    ├── tests/
    └── vars/
        └── main.yml
```

### Important Directories

| Directory    | Purpose                              |
| ------------ | ------------------------------------ |
| `tasks/`     | Main tasks performed by the role     |
| `handlers/`  | Actions triggered using `notify`     |
| `templates/` | Jinja2 templates                     |
| `files/`     | Static files copied to managed nodes |
| `vars/`      | Role variables                       |
| `defaults/`  | Default variables                    |
| `meta/`      | Role metadata and dependencies       |

---

# 3. Creating Roles Manually

Example project:

```text
playbooks/
├── master.yml
└── roles/
    ├── pkgs/
    │   └── tasks/
    │       └── main.yml
    ├── users/
    │   └── tasks/
    │       └── main.yml
    └── webserver/
        └── tasks/
            └── main.yml
```

Create the directories:

```bash
mkdir playbooks
cd playbooks

mkdir -p roles/pkgs/tasks
mkdir -p roles/users/tasks
mkdir -p roles/webserver/tasks
```

### Package Role

`roles/pkgs/tasks/main.yml`

```yaml
---
- name: Install packages
  yum:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - java-1.8.0-openjdk
    - tree
    - docker
    - maven
```

### User Role

`roles/users/tasks/main.yml`

```yaml
---
- name: Create users
  user:
    name: "{{ item }}"
    state: present
  with_items:
    - luckyy
    - imthiaz
    - siva
    - rajesh
    - pavan
```

### Webserver Role

`roles/webserver/tasks/main.yml`

```yaml
---
- name: Install webserver
  yum:
    name: httpd
    state: present

- name: Start httpd
  service:
    name: httpd
    state: started
```

### Master Playbook

```yaml
---
- name: ROLE PLAYBOOK
  hosts: all

  roles:
    - pkgs
    - users
    - webserver
```

Run:

```bash
ansible-playbook master.yml
```

---

# 4. Ansible Galaxy

Ansible Galaxy is a repository for sharing pre-built Ansible roles.

It allows you to:

* Search for existing roles
* Download roles
* Install roles
* Reuse community automation

Example:

```bash
ansible-galaxy role install criecm.tomcat
```

By default, roles are installed under:

```text
/root/.ansible/roles
```

You can configure another role path in `ansible.cfg`:

```ini
[defaults]
role_path = /etc/ansible/roles
```

---

# 5. Creating a Role with Ansible Galaxy

Instead of manually creating every directory, use:

```bash
cd /etc/ansible/roles

ansible-galaxy init Reyaz
```

This automatically creates the standard role structure.

```text
reyaz/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── README.md
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml
```

---

# 6. Nginx Role Project

Create the project:

```bash
mkdir nginx-role-project
cd nginx-role-project

ansible-galaxy init roles/nginx
```

## Variables

`roles/nginx/defaults/main.yml`

```yaml
nginx_package: nginx
nginx_service: nginx
nginx_port: 80
```

## Tasks

`roles/nginx/tasks/main.yml`

```yaml
---
- name: Install nginx
  ansible.builtin.package:
    name: "{{ nginx_package }}"
    state: present

- name: Start and enable nginx
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: started
    enabled: true

- name: Deploy custom index page
  ansible.builtin.template:
    src: index.html.j2
    dest: /usr/share/nginx/html/index.html
  notify: Restart nginx
```

## Handler

`roles/nginx/handlers/main.yml`

```yaml
---
- name: Restart nginx
  ansible.builtin.service:
    name: "{{ nginx_service }}"
    state: restarted
```

## Template

`roles/nginx/templates/index.html.j2`

```html
<!DOCTYPE html>
<html>
<head>
    <title>NGINX Server</title>
</head>
<body>
    <h1>Welcome to NGINX Server</h1>
    <p>Deployed using Ansible Role</p>
    <p>Running on port {{ nginx_port }}</p>
</body>
</html>
```

## Site Playbook

```yaml
---
- name: Install Nginx using Role
  hosts: prod
  become: true

  roles:
    - nginx
```

Run:

```bash
ansible-playbook site.yml
```

---

# 7. Ansible Vault

Ansible Vault is used to encrypt sensitive information.

Examples of sensitive information include:

* Passwords
* API keys
* SSH-related secrets
* Confidential variables

### Create Vault

```bash
ansible-vault create secret.txt
```

Example content:

```yaml
db_password: mysecurepassword
api_key: "12345-abcde-67890"
```

### Edit Vault

```bash
ansible-vault edit secret.txt
```

### Change Vault Password

```bash
ansible-vault rekey secret.txt
```

### Decrypt Vault

```bash
ansible-vault decrypt secret.txt
```

### Encrypt Again

```bash
ansible-vault encrypt secret.txt
```

### View Vault

```bash
ansible-vault view secret.txt
```

Ansible Vault helps protect sensitive automation data by keeping the stored content encrypted.

---

# 8. Asynchronous & Polling Actions

By default, Ansible executes tasks synchronously.

That means Ansible waits for one task to finish before continuing.

For long-running tasks, asynchronous execution can be used.

Example:

```yaml
---
- name: Async and poll playbook
  hosts: all
  ignore_errors: yes

  tasks:

    - name: Sleeping
      command: sleep 30
      async: 20
      poll: 10

    - name: Install git
      yum:
        name: git
        state: present
```

### Important Parameters

```yaml
async: 20
```

Maximum time allowed for the task.

```yaml
poll: 10
```

Ansible checks the task status every 10 seconds.

This approach is useful for long-running operations where you don't want the normal synchronous execution model.

---

# 9. Includes & Imports

As playbooks become large, maintaining everything in a single YAML file becomes difficult.

Ansible allows tasks and playbooks to be separated into multiple files.

Three important mechanisms are:

```text
import_tasks
include_tasks
import_playbook
```

---

# 10. import_tasks

`import_tasks` statically imports another task file.

Example:

```text
ansible/
├── site.yml
├── install.yml
└── configure.yml
```

### install.yml

```yaml
---
- name: Install Nginx
  dnf:
    name: nginx
    state: present
```

### configure.yml

```yaml
---
- name: Copy index.html
  copy:
    src: index.html
    dest: /usr/share/nginx/html/index.html
```

### site.yml

```yaml
---
- hosts: all
  become: yes

  tasks:
    - import_tasks: install.yml
    - import_tasks: configure.yml
```

Run:

```bash
ansible-playbook site.yml
```

### Key Point

`import_tasks` is **static**.

It is useful when the workflow is fixed and the same tasks should always be loaded.

---

# 11. include_tasks

`include_tasks` dynamically loads a task file during execution.

Example:

```yaml
---
- hosts: all
  become: yes

  vars:
    install_webserver: true

  tasks:
    - include_tasks: install.yml
      when: install_webserver
```

If:

```yaml
install_webserver: true
```

the task file is executed.

If:

```yaml
install_webserver: false
```

the task file is skipped.

### Example

```yaml
- include_tasks: nginx.yml
  when: webserver == "nginx"

- include_tasks: apache.yml
  when: webserver == "apache"
```

Only the matching task file is loaded.

---

# 12. import_playbook

`import_playbook` imports an entire playbook.

Example:

```text
install.yml
deploy.yml
backup.yml
master.yml
```

### master.yml

```yaml
---
- import_playbook: install.yml
- import_playbook: deploy.yml
- import_playbook: backup.yml
```

Run:

```bash
ansible-playbook master.yml
```

---

# 13. import_tasks vs include_tasks vs import_playbook

| Feature    | import_tasks    | include_tasks             | import_playbook             |
| ---------- | --------------- | ------------------------- | --------------------------- |
| Type       | Static          | Dynamic                   | Static                      |
| Imports    | Task file       | Task file                 | Entire playbook             |
| Loaded     | During parsing  | During execution          | During playbook parsing     |
| Conditions | Less flexible   | Good for conditions       | Playbook-level organization |
| Main use   | Fixed workflows | Dynamic/conditional tasks | Large playbook organization |

### Easy Way to Remember

```text
import_tasks
      ↓
Task File
      ↓
Static

include_tasks
      ↓
Task File
      ↓
Dynamic

import_playbook
      ↓
Entire Playbook
```

---

# 14. Ansible Collections

An Ansible Collection is a standardized bundle of Ansible automation content.

It can contain:

* Modules
* Roles
* Plugins
* Documentation
* Other automation content

Think of a Collection as a **toolbox for a particular technology**.

For example:

```text
amazon.aws
```

provides Ansible content for AWS automation.

---

# 15. Installing Ansible Collections

List installed collections:

```bash
ansible-galaxy collection list
```

Install the AWS collection:

```bash
ansible-galaxy collection install amazon.aws
```

The `amazon.aws` collection can then be used for AWS-related automation.

---

# 16. AWS S3 Automation with Ansible Collection

Install required Python libraries:

```bash
sudo dnf install python3-pip -y

pip3 install boto3 botocore
```

The notes use an IAM role for AWS access.

Example playbook:

```yaml
---
- name: S3 Collection Demo
  hosts: local
  gather_facts: false

  collections:
    - amazon.aws

  tasks:
    - name: Create S3 bucket
      amazon.aws.s3_bucket:
        name: dna-demo-bucket-unique-123
        state: present
        region: ap-south-1
```

Run:

```bash
ansible-playbook s3.yml
```

The example uses `hosts: local` so the AWS resource can be created from the Ansible machine rather than requiring the task to run against every managed host.

---

# 17. Overall Learning Summary

```text
                    ANSIBLE
                       |
       ┌───────────────┼────────────────┐
       |               |                |
     ROLES           VAULT          INCLUDES
       |               |                |
 Reusable          Secrets       Task Organization
 Automation        Encryption
       |
   GALAXY
       |
 Pre-built Roles
       |
 COLLECTIONS
       |
 Modules + Roles + Plugins
       |
     AWS
       |
   amazon.aws
       |
      S3
```

## Key Takeaways

1. **Roles** help organize automation into reusable components.
2. **Ansible Galaxy** provides reusable roles and automation content.
3. **Ansible Vault** protects sensitive information.
4. **Async & Polling** can be used for long-running tasks.
5. **import_tasks** statically imports task files.
6. **include_tasks** dynamically includes task files.
7. **import_playbook** imports an entire playbook.
8. **Collections** package Ansible automation content for specific technologies.
9. The **amazon.aws** collection can be used for AWS automation.
10. Ansible can be used to automate AWS resources such as S3.
