# Project-Ansible-Phonebook : Web Page Application (Mysql-Nodejs-Flask) deployed on EC2's with Ansible

## Learning Outcomes

At the end of this hands-on training, students will be able to;

- Explain how to create directory layout using Ansible

- Explain how to make error handling in Ansible

- Explain how to control playbook execution in Ansible

## Outline

- Part 1 - Build the Infrastructure (2 EC2 Instances with Ubuntu 20.04 AMI)

- Part 2 - Pinging the Target Nodes

- Part 3 - Install, Start, Enable Mysql and Run The Phonebook App

- Part 4 - File separation

- Part-5 - Error Handling and Controlling playbook execution: strategies

## Part 1 - Build the Infrastructure

- Get to the AWS Console and spin-up 2 EC2 Instances with ```Ubuntu 20.04``` AMI.

- Configure the security groups as shown below:

  - Controller Node ----> My Local Host

  - Target Node1 -------> Port 22 SSH, Port 3306 MYSQL/Aurora

  - Target Node2 -------> Port 22 SSH, Port 80 HTTP

## Part 2 - Pinging the Target Nodes

- Make a directory named ```ansible-phonebook``` under the home directory and cd into it.

```bash
mkdir ansible-phonebook
cd ansible-phonebook
```

- Copy the phonebook app files (`phonebook-app.py`, `requirements.txt`, `init.sql`, `templates`) to the control node from your github repository.

- Do not forget to change db server private ip in phonebook-app.py. (`app.config['MYSQL_DATABASE_HOST'] = "<db_server private ip>"`)

- Create a file named ```ping-playbook.yml``` and paste the content below.

```bash
touch ping-playbook.yml
```

```yml
- name: ping them all
  hosts: ansible_development
  tasks:
    - name: pinging
      ansible.builtin.ping:
```

- Run the command below for pinging the servers.

```bash
ansible-playbook ping-playbook.yml
```

- Explain the output of the above command.

## Part4 - Install, Start, Enable Mysql and Run The Phonebook App.

- Create a playbook name `mysql_config.yml` and configure db_server.

```yml
- name: db configuration
  become: true
  hosts: your db server name
  vars_files:
    - secret.yml
  vars:
    hostname: cw_db_server
    db_name: phonebook_db
    db_table: phonebook
    db_user: remoteUser

  tasks:
    - name: set hostname
      ansible.builtin.shell: "sudo hostnamectl set-hostname {{ hostname }}"  # hostname atama komutu

    - name: Installing Mysql  and dependencies
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop:
        - mysql-server
        - mysql-client
        - python3-mysqldb
        - libmysqlclient-dev

    - name: start and enable mysql service
      ansible.builtin.service:
        name: mysql
        state: started
        enabled: yes

    - name: creating mysql user
      community.mysql.mysql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        priv: '*.*:ALL'
        host: '%'
        state: present

    - name: copy the sql script
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-phonebook/phonebook/init.sql
        dest: ~/

    - name: creating phonebook_db
      community.mysql.mysql_db:
        name: "{{ db_name }}"
        state: present

    - name: check if the database has the table
      ansible.builtin.shell: |
        echo "USE {{ db_name }}; show tables like '{{ db_table }}'; " | mysql
      register: resultOfShowTables

    - name: DEBUG
      ansible.builtin.debug:
        var: resultOfShowTables

    - name: Import database table
      community.mysql.mysql_db:
        name: "{{ db_name }}"   # This is the database schema name.
        state: import  # This module is not idempotent when the state property value is import.
        target: ~/init.sql # This script creates the products table.
      when: resultOfShowTables.stdout == "" # This line checks if the table is already imported. If so this task doesn't run.

    - name: Enable remote login to mysql
      ansible.builtin.lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql

  handlers:
    - name: Restart mysql
      ansible.builtin.service:
        name: mysql
        state: restarted
```
- create secret.yml

```ansible-vault create secret.yml

New Vault password: xxxx
Confirm Nev Vault password: xxxx

```yml

db_password: clarus1234

```

- look at the content

```bash
cat secret.yml
```
```
33663233353162643530353634323061613431366332373334373066353263353864643630656338
6165373734333563393162333762386132333665353863610a303130346362343038646139613632
62633438623265656330326435646366363137373333613463313138333765366466663934646436
3833386437376337650a636339313535323264626365303031366534363039383935333133306264
61303433636266636331633734626336643466643735623135633361656131316463

- Run the playbook.

```bash
ansible-playbook --ask-vault-pass mysql_config.yml 
```

- Open up a new Terminal or Window and connect to the ```db_server``` instance and check if ```MariaDB``` is installed, started, and enabled.

```bash
mysql --version
```

- Or, you can do it with ad-hoc command.

```bash
ansible db_server -m shell -a "mysql --version"
```

- Create another playbook name `web_config.yml` and configure web_server.

```yml
- name: web server configuration
  hosts: your web server host name
  vars:
    hostname: cw_web_server
  tasks:
    - name: set hostname
      ansible.builtin.shell: "sudo hostnamectl set-hostname {{ hostname }}"

    - name: Installing python for python app
      become: yes
      ansible.builtin.package:
        name:
          - python3
          - python3-pip
        state: present
        update_cache: yes

    - name: copy the app file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-phonebook/phonebook/phonebook-app.py
        dest: ~/

    - name: copy the requirements file to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-phonebook/phonebook/requirements.txt
        dest: ~/

    - name: copy the templates folder to the web server
      ansible.builtin.copy:
        src: /home/ubuntu/ansible-phonebook/phonebook/templates
        dest: ~/

    - name: install dependencies from requirements file
      become: yes
      ansible.builtin.pip:
        requirements: /home/ubuntu/requirements.txt

    - name: run the app
      become: yes
      ansible.builtin.shell: "nohup python3 phonebook-app.py &"
```

- Run the playbook.

```bash
ansible-playbook web_config.yml
```

- Check if you can see the website on your browser.
