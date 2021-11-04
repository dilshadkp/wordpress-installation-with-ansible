# Ansible Playbook to host a Wordpress site in a LAMPStack

## Description

This is an Ansible Playbook to host a Wordpress website in a LAMPStack.


## Prerequisites

- Install Ansible in Ansible Master server. [Click here for Ansible Installation steps](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

- Configure Inventory/Hosts file


>Configure inventory file in Ansible Master and add the client server details to inventory as below.

```
[Redhat]
IP-of-client-server	    ansible_user="SSH-user"		ansible_port="SSH-port-of-client"	    ansible_private_key_file="/path/to/ssh-key.pem"
```
>Note that this SSH user should have sudo privileges in the client server without a password.
>You can add the user to the sudoes list without password by executing below command in client server as root:

```shell
echo "<SSH-user> ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```
- ## Ansible Modules used in this Playbook
  - [yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html)
  - [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
  - [service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
  - [shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
  - [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
  - [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
  - [mysql_user](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html)
  - [mysql_db](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html)
  - [get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)
  - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)
  
- [with_items](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html) is used as a loop to restart services

- ## Features of this Playbook
  - Used tags for each tasks so that you can control tasks based on tags.
  - You can provide your preferred domain name in ***vhost.vars*** file. The wordpress will be installed under that domain in the server.


- ## Tasks defined in the Playbook


 - #### Task 1

>These tasks will perform below actions:
- Install Apache service in the server
- Install PHP 7.4 in the server

```python
   - name: "Install apache"
      yum:
        name: httpd
        state: present
      tags:
        - lamp
        - apache

   - name: "Install PHP 7.4"
      shell: amazon-linux-extras install php7.4 -y
      tags:
        - lamp
        - apache
```
 - #### Task 2

>These task wil perform below actions:
- copy the content of template *httpd.conf.tmpl* in the master machine to */etc/httpd/conf/httpd.conf* in the client server. By doing this, we can control the apache configuration through Ansible.
- Copy the content of template *vhost.conf.tmpl* in the master machine to */etc/httpd/conf.d/{{http_domain}}.conf* in the client server. By doing this, we can control the virtual host configuration through Ansible.

```python
    - name: "copy httpd conf from template"
      template:
        src: httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
      tags:
        - lamp
        - apache
    - name: "copy virtualhost from template"
      template:
        src: vhost.conf.tmpl
        dest: /etc/httpd/conf.d/{{http_domain}}.conf
      tags:
        - lamp
        - apache
```
 - #### Task 3

>These task wil perform below actions:
- Create document root for the site and set proper ownerships to the directory
- Create a phpinfo() page in the document root. This is not a compulsory task

```python
    - name: "create document root for website"
      file:
        path: /var/www/html/{{http_domain}}/
        state: directory
        owner: "{{http_user}}"
        group:  "{{http_group}}"
      tags:
        - lamp
        - apache
    - name: "create phpinfo page"
      copy:
        content: "<?php phpinfo(); ?>"
        dest: /var/www/html/{{http_domain}}/phpinfo.php
        owner: "{{http_user}}"
        group: "{{http_group}}"
      tags:
        - lamp
        - apache
```
 - #### Task 4

>These task wil perform below actions:
- Install Mariadb-server and MySQL-python packages in the server.
- Restart and enable HTTPD and Mariadb services.


```python
    - name: "Install mariadb"
      yum:
        name: 
          - mariadb-server
          - MySQL-python
        state: present
      tags:
        - lamp
        - mariadb
    - name: "Restart and enable LAMP services"
      service: 
        name: "{{item}}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb
      tags:
        - lamp
        - mariadb
```
 - #### Task 5

>These task wil perform below actions:
- Set password for mysql root user
- Remove anonymouse users
- Remove test database.

```python
    - name: "setting root password for mariadb"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{db_root_password}}"
        host_all: yes
      tags:
        - lamp
        - mariadb

    - name: "remove anonymous users"
      mysql_user:
        login_user: "root"
        login_password: "{{db_root_password}}"
        user: ""
        state: absent
      tags:
        - lamp
        - mariadb

    - name: "remove test database"
      mysql_db:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "test"
        state: absent
      tags:
        - lamp
        - mariadb
```
 - #### Task 6

>These task wil perform below actions:
- Create a database for Wordpress
- Create a db user for the wordpress site and grant all privileges on wordpress database to this user

```python
    - name: "create wordpress database"
      mysql_db:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "{{wp_db}}"
        state: present
      tags:
        - lamp
        - mariadb

    - name: "create wordpress db user"
      mysql_user:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "{{wp_db_user}}"
        state: present
        password: "{{wp_db_user_password}}"
        priv: '{{wp_db}}.*:ALL'
      tags:
        - lamp
        - mariadb
```

 - #### Task 7

>These set of tasks will perform below actions:
- Download latest version of Wordpress from Internet to /tmp of client server
- Extract the tar.gz file to /tmp
- Copy the wordpress content from /tmp to document root of the website
- Copy the wp-config.php.tmpl template from master to document root of website. In this way, we can control wordpress configuration file through Ansible
- Clean up directories under /tmp
- Restart HTTPD and Mariadb services

```python
    - name: "Download wordpress from Internet"
      get_url:
        url: "{{download_link}}"
        dest: "/tmp/wordpress.tar.gz"
        remote_src: true
      tags:
        - wordpress

    - name: "extract tar file"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true
      tags:
        - wordpress

    - name: "copy wordpress files from /tmp to docroot"
      copy: 
        src: "/tmp/wordpress/"
        dest: /var/www/html/{{http_domain}}/
        owner: "{{http_user}}"
        group: "{{http_group}}"
        remote_src: true
      tags:
        - wordpress

    - name: "configuring wp-config.php from template"
      template:
        src: wp-config.php.tmpl
        dest: /var/www/html/{{http_domain}}/wp-config.php
        owner: "{{http_user}}"
        group: "{{http_group}}"
      tags:
        - wordpress

    - name: "clean-up unwanted files"
      file:
        path: "{{item}}"
        state: absent
      with_items:
        - "/tmp/wordpress"
        - "/tmp/wordpress.tar.gz"
      tags:
        - wordpress

    - name: "Final restart of services"
      service:
        name: "{{item}}"
        state: restarted
      with_items:
        - httpd
        - mariadb
      tags:
        - wordpress
```

- ## Execution
 - Put both the Playbook and inventory file along with the SSH key to access client server in working directory in the Ansible Master server.
 - Run a syntax check

 ```bash
ansible-playbook -i inventory lamp-in-redhat.yml --syntax-check
```
 - Execute the Playbook

 ```bash
ansible-playbook -i inventory lamp-in-redhat.yml
```
- You can control the execution to specific tasks using [**tags**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html)

Example:
Below command will execute only the tasks with tag **lamp**
 ```bash
ansible-playbook -i inventory lamp-in-redhat.yml --tags lamp
```

x--------------------x---------------------x---------------------x---------------------x---------------------x---------------------x---------------------x---------------------x
### ⚙️ Connect with Me 

<p align="center">
<a href="mailto:dilshad.lalu@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dilshadkp/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a> 
<a href="https://www.instagram.com/dilshad_a.k.a_lalu/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://wa.me/%2B919567344212?text=This%20message%20from%20GitHub."><img src="https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white"/></a><br />
