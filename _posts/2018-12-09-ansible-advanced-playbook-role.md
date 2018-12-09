---
layout: post
title: "DevOps - Ansible: Advanced Playbooks and Role Development"
date: 2018-12-09 10:25:56 +01
categories: devops ansible
tags: ansible automation
---

## Intro

Ansible's power of automation, and its capabilities extend far beyond basic playbooks. Advanced playbook features and role development allow you to create modular, reusable, and highly efficient automation workflows. This guide explores **advanced concepts** such as conditionals, strategies, custom roles, and dynamic task execution.

---

## Step 1: Advanced Playbook Features

### **1.1 Using Conditionals**

Conditionals enable tasks to run only when specific conditions are met.

#### Example:

Check if a service is running before attempting to restart it:

```yml
- name: Check if Apache is running
  ansible.builtin.shell: systemctl is-active apache2
  register: apache_status
  ignore_errors: yes

- name: Restart Apache if it is running
  ansible.builtin.service:
    name: apache2
    state: restarted
  when: apache_status.rc == 0
```

### **1.2 Delegation and Local Actions**

Run tasks on a different host or locally on the control node.

#### Example:

Generate an SSL certificate locally and copy it to the target server:

```yml
- name: Generate SSL certificate locally
  ansible.builtin.command:
    cmd: openssl req -new -x509 -days 365 -nodes -out /tmp/cert.pem -keyout /tmp/key.pem
  delegate_to: localhost

- name: Copy SSL certificate to the server
  ansible.builtin.copy:
    src: /tmp/cert.pem
    dest: /etc/ssl/certs/cert.pem
```

### **1.3 Free Strategy for Parallel Execution**

Use the `free` strategy to allow tasks to run independently on different hosts.

#### Example:

```yml
- name: Deploy Web Application with Free Strategy
  hosts: web_servers
  strategy: free
  tasks:
    - name: Install dependencies
      ansible.builtin.yum:
        name: httpd
        state: latest

    - name: Start web server
      ansible.builtin.service:
        name: httpd
        state: started
```

---

## Step 2: Developing Custom Roles

Roles provide a structured way to organize playbooks into reusable components.

### **2.1 Creating a Role**

Generate a role skeleton using `ansible-galaxy`:

```yml
ansible-galaxy init webserver_role
```

This creates the following directory structure:

```sh
webserver_role/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── templates/
├── files/
├── vars/
│   └── main.yml
├── defaults/
│   └── main.yml
├── meta/
│   └── main.yml
```

### **2.2 Writing Tasks for the Role**

Define tasks in `tasks/main.yml`:

```yml
- name: Install Apache web server
  ansible.builtin.yum:
    name: httpd
    state: present

- name: Start Apache service
  ansible.builtin.service:
    name: httpd
    state: started

- name: Deploy index.html from template
  ansible.builtin.template:
    src: index.html.j2
    dest: /var/www/html/index.html

- name: Notify restart of Apache service if configuration changes
  notify:
    - Restart Apache Service
```

### **2.3 Adding Handlers**

Define handlers in `handlers/main.yml`:

```yml
- name: Restart Apache Service
  ansible.builtin.service:
    name: httpd
    state: restarted
```

### **2.4 Using the Role in a Playbook**

Reference the role in your playbook:

```yml
- hosts: web_servers
  roles:
    - role: webserver_role
```

---

## Step 3: Dynamic Task Execution

Dynamic task execution enables flexible workflows based on runtime variables or conditions.

### **3.1 Looping Over Dynamic Variables**

Iterate over a list of users to create accounts dynamically.

```yml
- name: Create users dynamically from a list of variables
  ansible.builtin.user:
    name: "{{ item.name }}"
    state: present
    shell: "{{ item.shell }}"
  loop:
    - { name: "alice", shell: "/bin/bash" }
    - { name: "bob", shell: "/bin/zsh" }
```

### **3.2 Include Tasks Dynamically**

Include tasks based on conditions.

```yml
- name: Include tasks for Debian-based systems only
  include_tasks: debian_tasks.yml
  when: ansible_os_family == "Debian"

- name: Include tasks for RedHat-based systems only
  include_tasks: redhat_tasks.yml
  when: ansible_os_family == "RedHat"
```

---

## Step 4: Best Practices for Roles and Playbooks

1. **Follow Separation of Concerns**  
   Keep roles single-purposed (e.g., separate roles for web servers, databases, etc.).

2. **Use Descriptive Names**  
   Use meaningful names for roles, tasks, and variables to improve readability.

3. **Document Roles**  
   Provide clear documentation for required variables and role functionality.

4. **Avoid Hardcoding Values**  
   Use default variables in `defaults/main.yml` and allow overrides via `vars`.

5. **Test Roles Thoroughly**  
   Use tools like Molecule to test your roles in isolated environments.

---

## Step 5: Advanced Role Usage with Parameters

Pass parameters to roles for flexibility.

#### Example Playbook with Parameterized Role:

```yml
- hosts: all
  roles:
    - role: webserver_role
      vars:
        http_port: 8080
        server_name: example.com
```

In the role's `tasks/main.yml`, use these variables dynamically:

```yml
- name: Configure Apache virtual host file from template
  ansible.builtin.template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/{{ server_name }}.conf
```

---

## Conclusion

Advanced Ansible playbooks and role development unlock powerful automation capabilities that are reusable, modular, and maintainable. By using features like conditionals, dynamic task execution, free strategies, and custom roles, you can create robust workflows tailored to complex environments. Follow best practices to ensure your Ansible projects remain scalable and easy to manage.
