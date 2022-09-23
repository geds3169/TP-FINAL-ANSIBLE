<h1 align="center">TP-FINAL-ANSIBLE</h1>  

<h2 align="center">Création de playbooks pour l'installation de MySQL, httpd, PHP7.4, Wordpress</h2>  

**<u>Groupe de travail:</u>**  

    - Christophe GARCIA            - David LAGNEAU            - Emmanuel GIRIN                
    - Madj EL KHATIB               - Magatte DIOP             - Guilhem SCHLOSSER    

### <u>Instruction du TP</u>  

```ini
. Un playbook qui s'appellera php.yml qui préparera l'environnement en installant PHP, Apache et tout le nécessaire
    - Installation Apache
    - Installation php

. Un playbook qui s'appellera wordpress.yml qui téléchargera Wordpress et mettra en place la configuration de wordpress
    - Installation du wordpress
    - Configuration d'Apache

. Un playbook qui s'appellera mysql.yml qui installera Mysql et va mettre en place la configuration necessaire pour la base de données
    - Installation de mysql 
    - Configuration de mysql

. un playbook qui s'appellera master.yml qui déploiera Wordpress en appelant les autres playbooks
```        

### <u>Presrequis:</u>

Pour ce TP, nous avons utilisé Vagrant, nous a été un dossier contenant l'ensemble des outils nécessaire à la mise en place de nos infrastructure (labs)

Nous avons déployer un Ansible master ainsi que deux hôtes.

Création de l'arborescence du projet

```bash
mkdir -p <nom_du_projet>
cd <nom_du_projet>
mkdir -p env-inventory
mkdir -p group_vars
mkdir -p host_vars
mkdir -p roles
```

un fichier hosts.yaml qui contient les informations de nos hotes (clients)

```bash
vim hosts.yaml
```

```yaml
---
# hosts.yaml

all:
  children:
    prod:
      hosts:
        client1:
          ansible_host: 192.168.99.11
    dev:
      hosts:
        client2:
          ansible_host: 192.168.99.12
    database:
      hosts:
        client3:
          ansible_host: 192.168.99.13
    webservers:
      hosts:
        client4:
          ansible_host: 192.168.99.14    
```

```bash
vim host_vars/prod.yml
```

```yml
# prod.yml
all:
  children:
    prod:
      hosts:
        client1:
```

```bash
vim host_vars/dev.yml
```

```yml
# prod.yml
all:
  children:
    dev:
      hosts:
        client2:
```

vim group_vars/prod.yaml

```
---
ansible_user: vagrant
```

vim group_vars/dev.yaml

```yaml
---
ansible_user: vagrant
```    

---
PARTIE 1
---
        
<h3 align="center">Création des différents playbooks</h3>  

**MySQL**

Création d'un playbook pour l'installation de MySQL mysql.yml

```yaml
---
# mysql.yml

- name: "Installation de mysql"
  hosts: prod
  become: yes
  vars_files:
      - files/secrets/credentials.yml
  vars:
      mysql_user_login: wordpress
      mysql_root_password: "R00T_P@SS0rD"
      mysql_user_password: wordpress
      ansible_python_interpreter: /usr/bin/python3
      db_name: wordpress
  pre_tasks:
      - name: "Installation du repo"
        yum:
           name: http://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
           state: present
      - name: "Import gpg key"
        rpm_key:
            state: present
            key: https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
      - name: "Install python3-PyMySQL library"
        ansible.builtin.pip:
          name:  PyMySQL
          state: present
  #        executable: pip3
  tasks:
    - name: "Install MySQL Server"
      yum:
        name: mysql-community-server
      register: fresh_install
      when: ansible_os_family == "RedHat"
    - name: "Start MySQL Server"
      service:
        name: mysqld
        state: started
        enabled: yes
    - name: Get temporary MySQL root password
      shell: grep 'temporary password' /var/log/mysqld.log | awk '{print $NF}'
      when: fresh_install.changed
      register: mysql_root_temp_password
      #mysql_root_temp_password.stdout

    - name: "Init root password"
      shell: mysql -u root --connect-expired-password --password="{{mysql_root_temp_password.stdout}}" -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{mysql_root_password}}';"
      when: fresh_install.changed

    - name: "Start MySQL Server"
      service:
        name: mysqld
        state: restarted
        enabled: yes

    - name: check if user exists
      mysql_user:
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        name: "{{ mysql_user_login }}"
        state: absent
      register: user_absent

    - name: "Create a new database with name 'wordpress'"
      mysql_db:
        name: "{{db_name}}"
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        state: present
        encoding: utf8mb4
        collation: utf8mb4_unicode_ci
      when: user_absent

    - name: Create MY_DBA user in MySQL and grant privileges
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root_password }}"
        user: "{{ mysql_user_login }} "
        password: "{{ mysql_user_password }}"
        host: 'localhost'
        priv: "{{db_name}}.*:ALL"  host: 'localhost'
        priv: "{{db_name}}.*:ALL"
```      

**Apache (httpd) + PHP 7.4**

Création d'un playboook pour l'installation de apache (httpd) apache.yml

```yaml
---
# apache.yml

- name: Install apache and php7 Centos 7 machine
  hosts: all
  become: yes
  vars_files:
      - files/secrets/credentials.yml
  pre_tasks:
    - name: "Check if Firewalld is absent"
      package:
         name: "firewalld"
         state: absent
      register: install_firewalld

    - name: "Check if epel-release is absent"
      package:
         name: epel-release
         state: absent
      register: install_epel

    - name: "Installation firewalld if not present"
      yum:
         name: firewalld
         state: present
      when: install_firewalld

    - name: "Start and enable firewalld service"
      service:
         name: firewalld
         enabled: yes
         state: started
    - name: "Install the remi repo"
      yum:
         name: https://rpms.remirepo.net/enterprise/remi-release-7.rpm
         state: present
      when: install_epel
    - name: "Import gpg key"
      rpm_key:
         state: present
         key: https://rpms.remirepo.net/RPM-GPG-KEY-remi
  tasks:
    - name: "Install httpd"
      yum:
        name: httpd
        state: present
    - name: "Start httpd"
      service:
        name: httpd
        state: started
        enabled: yes
    - name: "permit traffic in public zone for https service"
      firewalld:
        service: https
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "permit traffic in public zone for http service"
      firewalld:
        service: http
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "Install php 7.4 and necessary extensions"
      yum:
        name:
          - php
          - php-bz2
          - php-mysql
          - php-curl
          - php-gd
          - php-intl
          - php-common
          - php-mbstring
          - php-xml
        enablerepo: remi-php74
        update_cache: true

    - name: "Restart apache"
      service:
        name: httpd
        state: restarted
```

Création d'un template pour la configuration automatisé d'apache

virtual_host.j2    

```jinja2
<VirtualHost *:{{ port }}>
   ServerAdmin ***********************
   ServerName {{ host }}
   ServerAlias www.{{ host }}
   DocumentRoot /var/www/{{ host }}/wordpress
   ErrorLog /var/log/httpd/error.log
   CustomLog /var/log/httpd/access.log combined

   <Directory /var/www/{{ host }}/wordpress>
         Options Indexes FollowSymLinks
     AllowOverride all
     Require all granted
   </Directory>
</VirtualHost>
```      

**Wordpress**

Création d'un fichier pour l'installation du Wordpress, wordpress.yml

```yaml
---
---
# wordpress.yml

- name: Install wordpress en Centos 7 machine
  hosts: all
  become: yes
  vars_files:
      - files/secrets/credentials.yml
  vars:
     host: myhostname
     port: 80
     db_name: wordpress
     mysql_user_login: wordpress
     mysql_user_password: wordpress
     db_host: localhost
  tasks:
  - name: "Create dir /var/www/{{host}}"
    file:
       path: "/var/www/{{ host }}"
       state: directory
       owner: apache
       group: apache
       mode: "755"

  - name: Téléchargement de la version latest WordPress
    unarchive:
       src: https://wordpress.org/latest.tar.gz
       dest: "/var/www/{{ host }}"
       remote_src: yes
       owner: apache
       group: apache
       mode: "755"

  - name: Mettre en place Apache VirtualHost
    template:
       src: "files/virtual_host.j2"
       dest: "/etc/httpd/conf.d/{{ host }}"
       owner: root
       group: root
       mode: '0755'

  - name: Mettre en place Apache VirtualHost
    template:
       src: "files/wp-config.php.j2"
       dest: "/var/www/{{ host }}/wordpress/wp-config.php"
       owner: root
       group: root
       mode: '0755'

  - name: "restart httpd"
    service:
       name: httpd
       state: restarted
       enabled: yes
```

Création d'un template pour la configuration automatisé de wordpress

wp-config.php.j2

```jinja2
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{db_name}}' );

/** Database username */
define( 'DB_USER', '{{mysql_user_login}}' );

/** Database password */
define( 'DB_PASSWORD', {{'mysql_user_password'}} );

/** Database hostname */
define( 'DB_HOST', {{db_host}} );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', '' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'SECURE_AUTH_KEY',  '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'LOGGED_IN_KEY',    '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'NONCE_KEY',        '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'AUTH_SALT',        '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'SECURE_AUTH_SALT', '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'LOGGED_IN_SALT',   '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'NONCE_SALT',       '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```

**Playbook master**

Création d'un fichier master.yml pour importer les différents playbooks à jouer.

```yaml
---

# Playbook d'installation d'un site web complet basé sur Wordpress

- name: "Installation de PHP et Apache"
  import_playbook: php.yml
  tags: php-apache
- name: "installation de  Mysql et la configuration necessaire "
  import_playbook: mysql.yml
  tags: mysql
- name: "téléchargement de Wordpress et sa  configuration "
  import_playbook: wordpress.yml
  tags: wordpress
```

Création d'un repertoire et fichier qui contiendront les secret (password) de l'utilisateur, sudo.

```bash
cd <nom_du_projet>
mkdir -p files/secrets/credentials.yml
```

Edition du fichier credentials.yml

```bash
vim files/secrets/credentials.yml
```

```yaml
ansible_sudo_pass_client1: vagrant
ansible_sudo_pass_client2: vagrant
```

**Jouer le playbook**

```bash
ansible-playbook -i hosts.yml master.yml
```

---
PARTIE 2
---

<h2 align="center">Modification des playbooks pour l'installation de MySQL, httpd, PHP7.4, Wordpress en roles</h2>


création des différents répertoire de rôle dans le répertoire du projet

```bash
ansible-galaxy init MySQL
```

```bash
ansible-galaxy init Apache
```

```bash
ansible-galaxy init php7.4
```

```bash
ansible-galaxy init Wordpress
```

Modification des différents playbooks afin de les transformer en rôles, déplacement des différents fichier (.yaml) dans leur repertoire rôle respectif

**MySQL**

main.yml

```yaml
---
# tasks file for mysql
- import_tasks: install_requirements.yml
- import_tasks: mysql.yml
```

install_requirements.yml

```yaml
---
      - name: "Installation du repo"
        yum:
           name: http://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
           state: present
      - name: "Import gpg key"
        rpm_key:
            state: present
            key: https://repo.mysql.com/RPM-GPG-KEY-mysql-2022

      - name: 
        yum:
          name: python3-pip
          state: present

      - name: "Install python3-PyMySQL library"
        ansible.builtin.pip:
          name:  PyMySQL
          state: present
          executable: pip3
```

mysql.yml

```yaml
    - name: "Install MySQL Server"
      yum:
        name: mysql-community-server
      register: fresh_install
      when: ansible_os_family == "RedHat"

    - name: "Start MySQL Server"
      service:
        name: mysqld
        state: started
        enabled: yes

    - name: Get temporary MySQL root password
      shell: grep 'temporary password' /var/log/mysqld.log | awk '{print $NF}'
      when: fresh_install.changed
      register: mysql_root_temp_password
      #mysql_root_temp_password.stdout

    - name: "Init root password"
      shell: mysql -u root --connect-expired-password --password="{{mysql_root_temp_password.stdout}}" -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{mysql_root_password}}';"
      when: fresh_install.changed

    - name: "Start MySQL Server"
      service:
        name: mysqld
        state: restarted
        enabled: yes

    - name: check if user exists
      mysql_user:
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        name: "{{ mysql_user_login }}"
        state: absent
      register: user_absent


    - name: "Create a new database with name 'wordpress'"
      mysql_db:
        name: "{{db_name}}"
        login_user: "root"
        login_password: "{{ mysql_root_password }}"
        state: present
        encoding: utf8mb4
        collation: utf8mb4_unicode_ci
      when: user_absent

    - name: Create MY_DBA user in MySQL and grant privileges
      mysql_user:
        login_user: root
        login_password: "{{ mysql_root_password }}"
        user: "{{ mysql_user_login }} "
        password: "{{ mysql_user_password }}"
        host: 'localhost'
        priv: "{{db_name}}.*:ALL"
```

Ajout des logins et password correspondant aux différentes variables, ainsi que la variable d'environnement et le nom de la base de donnée.

```bash
vim roles/mysql/defaults/main.yml
```

```yaml
---
# defaults file for mysql
mysql_user_login: wordpress
mysql_user_password: wordpress
mysql_root_password: "R00T_P@SS0rD"
db_name: wordpress
ansible_python_interpreter: /usr/bin/python3
```

**Apache (httpd)**

```bash
vim roles/php/tasks/install_requirements.yml
```

```yaml
---
# task Apache (httpd)

    - name: "Check if Firewalld is absent"
      package:
         name: "firewalld"
         state: absent
      register: install_firewalld

    - name: "Check if epel-release is absent"
      package:
         name: epel-release
         state: absent
      register: install_epel

    - name: "Installation firewalld if not present"
      yum:
         name: firewalld
         state: present
      when: install_firewalld

    - name: "Start and enable firewalld service"
      service:
         name: firewalld
         enabled: yes
         state: started
    - name: "Install the remi repo"
      yum:
         name: https://rpms.remirepo.net/enterprise/remi-release-7.rpm
         state: present
      when: install_epel
    - name: "Import gpg key"
      rpm_key:
         state: present
         key: https://rpms.remirepo.net/RPM-GPG-KEY-remi
```

```bash
vim projet_ansible_wordpress/roles/php/tasks/main.yml
```

```yaml
---
# tasks file for php
- import_tasks: install_requirements.yml
- import_tasks: php.yml
```

```bash
vim roles/php/tasks/php.yml
```

```yaml
---
    - name: "Install httpd"
      yum:
        name: httpd
        state: present
    - name: "Start httpd"
      service:
        name: httpd
        state: started
        enabled: yes
    - name: "permit traffic in public zone for https service"
      firewalld:
        service: https
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "permit traffic in public zone for http service"
      firewalld:
        service: http
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "Install php 7.4 and necessary extensions"
      yum:
        name:
          - php
          - php-bz2
          - php-mysql
          - php-curl
          - php-gd
          - php-intl
          - php-common
          - php-mbstring
          - php-xml
        enablerepo: remi-php74
        update_cache: true

    - name: "Restart apache"
      service:
        name: httpd
        state: restarted
```

```bash
vim roles/php/tasks/php.yml
```

```yaml
---
    - name: "Install httpd"
      yum:
        name: httpd
        state: present
    - name: "Start httpd"
      service:
        name: httpd
        state: started
        enabled: yes
    - name: "permit traffic in public zone for https service"
      firewalld:
        service: https
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "permit traffic in public zone for http service"
      firewalld:
        service: http
        zone: public
        permanent: yes
        immediate: yes
        state: enabled
    - name: "Install php 7.4 and necessary extensions"
      yum:
        name:
          - php
          - php-bz2
          - php-mysql
          - php-curl
          - php-gd
          - php-intl
          - php-common
          - php-mbstring
          - php-xml
        enablerepo: remi-php74
        update_cache: true

    - name: "Restart apache"
      service:
        name: httpd
        state: restarted
```

```bash
vim tasks/install_requirements.yml
```

```bash
---
    - name: "Check if Firewalld is absent"
      package:
         name: "firewalld"
         state: absent
      register: install_firewalld

    - name: "Check if epel-release is absent"
      package:
         name: epel-release
         state: absent
      register: install_epel

    - name: "Installation firewalld if not present"
      yum:
         name: firewalld
         state: present
      when: install_firewalld

    - name: "Start and enable firewalld service"
      service:
         name: firewalld
         enabled: yes
         state: started
    - name: "Install the remi repo"
      yum:
         name: https://rpms.remirepo.net/enterprise/remi-release-7.rpm
         state: present
      when: install_epel
    - name: "Import gpg key"
      rpm_key:
         state: present
         key: https://rpms.remirepo.net/RPM-GPG-KEY-remi
```

**Wordpress**

```bash
vim roles/wordpress/tasks/main.yml
```

```yaml
---
# tasks file for wordpress
- import_tasks: wordpress.yml
```

```bash
vim roles/wordpress/tasks/wordpress.yml
```

```yaml
---
  - name: "Create dir /var/www/html"
    file:
       path: "/var/www/{{ wordpress_host }}"
       state: directory
       owner: apache
       group: apache
       mode: "755"

  - name: Téléchargement de la version latest WordPress
    unarchive:
       src: https://wordpress.org/latest.tar.gz
       dest: "/var/www/html"
       remote_src: yes
       owner: apache
       group: apache
       mode: "755"

  - name: Mettre en place Apache VirtualHost
    template:
       src: "virtual_host.j2"
       dest: "/etc/httpd/conf.d/{{ wordpress_host }}"
       owner: root
       group: root
       mode: '0755'

  - name: Mettre en place Apache VirtualHost
    template:
       src: "wp-config.php.j2"
       dest: "/var/www/html/wordpress/wp-config.php"
       owner: root
       group: root
       mode: '0755'

  - name: "restart httpd"
    service:
       name: httpd
       state: restarted
       enabled: yes
```

```bash
vim roles/wordpress/templates/virtual_host.j2
```

```jinja2
<VirtualHost *:{{ wordpress_port }}>
   ServerName {{ wordpress_host }}
   ServerAlias www.{{ wordpress_host }}
   DocumentRoot /var/www/html/wordpress
   ErrorLog /var/log/httpd/error.log
   CustomLog /var/log/httpd/access.log combined

   <Directory /var/www/html/wordpress>
         Options Indexes FollowSymLinks
     AllowOverride all
     Require all granted
   </Directory>
</VirtualHost>
```

```bash
vim roles/wordpress/templates/wp-config.php.j2
```

```jinja2
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{db_name}}' );

/** Database username */
define( 'DB_USER', '{{mysql_user_login}}' );

/** Database password */
define( 'DB_PASSWORD', {{mysql_user_password}} );

/** Database hostname */
define( 'DB_HOST', {{db_host}} );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', '' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'SECURE_AUTH_KEY',  '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'LOGGED_IN_KEY',    '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'NONCE_KEY',        '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'AUTH_SALT',        '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'SECURE_AUTH_SALT', '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'LOGGED_IN_SALT',   '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );
define( 'NONCE_SALT',       '{{ lookup('password', '/dev/null chars=ascii_letters length=64') }}' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```

```bash
vim roles/wordpress/defaults/main.yml
```

```yaml
---
# defaults file for wordpress
wordpress_host: myhostname
wordpress_port: 80
db_name: wordpress
mysql_user_login: wordpress
mysql_user_password: wordpress
db_host: localhost
ansible_python_interpreter: /usr/bin/python
```

Edition du playbook

```yaml
# Installation complète d'un site internet, apache (httpd), mysql community server 5.7, wordpress

- name: "installation de wordpress"
  hosts: all
  become: true
  vars_files:
  #  - files/default.yml
    - files/secrets/credentials.yml
  roles:
    - role: php
    - role: mysql
    - role: wordpress
```

**Jouer le playbook**

```bash
ansible-playbook -i hosts.yml master.yml
```
