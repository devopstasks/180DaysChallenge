# Install Nagios
Use Ansible to install Nagios from documentation over [here](https://support.nagios.com/kb/article/nagios-core-installing-nagios-core-from-source-96.html#Ubuntu) for ubuntu and over [here](https://support.nagios.com/kb/article/nagios-core-installing-nagios-core-from-source-96.html#CentOS) for centos
Submission Process: 
Please write this ansible playbook in you GitHub Account and share the Git hub url on the slack channel 90 or 180 days once you have finished, We will have a zoom meeting with a reviewer.
The below section contains the crude way of installing nagios in ubuntu, make it work for centos as well and try to optimize the below playbook
---
- hosts: all
  vars:
    nagios_version: '4.4.5'
    nagios_plugin_version: '2.2.1'
    packages_ubuntu:
      - autoconf 
      - gcc 
      - libc6 
      - make 
      - wget 
      - unzip 
      - apache2 
      - php 
      - libapache2-mod-php7.2 
      - libgd-dev
      - libmcrypt-dev 
      - libssl-dev 
      - bc 
      - gawk 
      - dc 
      - build-essential 
      - snmp 
      - libnet-snmp-perl 
      - gettext
    nagios_code_location: /tmp/nagioscore.tar.gz
    nagios_plugin_location: /tmp/nagios-plugins.tar.gz
  become: yes
  tasks:
    - name: fail for unsupported os
      fail:
        msg: This playbook runs only on ubuntu
      when: ansible_os_family != 'debian'
    - name: install packages for ubuntu
      apt:
        name: "{{ item }}"
        update_cache: yes
        state: present
      loop: "{{ packages_ubuntu }}"
      tags:
        - nagios
    - name: Download the source to temp folder
      get_url:
        dest: "{{ nagios_code_location }}"
        url: "https://github.com/NagiosEnterprises/nagioscore/archive/nagios-{{ nagios_version }}.tar.gz"
      tags:
        - nagios
    - name: untar the code
      unarchive:
        src: "{{ nagios_code_location }}"
        dest: "/tmp/"
      tags:
        - nagios
    - name: Compile Code
      shell:
        cmd: sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled > configure.log && sudo make all > makeoutput.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Create User and Group
      shell:
        cmd: sudo make install-groups-users > installgroups.log && sudo usermod -a -G nagios www-data > usermod.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Binaries
      shell:
        cmd: sudo make install > makeinstall.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Service / Daemon
      shell:
        cmd: sudo make install-daemoninit > daemoninit.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Command Mode
      shell:
        cmd: sudo make install-commandmode > commandmode.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Configuration Files
      shell:
        cmd: sudo make install-config > config.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install Apache Config Files
      shell:
        cmd: sudo make install-webconf > webconf.log && sudo a2enmod rewrite > a2enmodrewrite.log && sudo a2enmod cgi > cgi.log
      args:
        chdir: "/tmp/nagioscore-nagios-{{ nagios_version }}/"
      tags:
        - nagios
    - name: Install latest passlib with pip
      pip: name=passlib
      tags:
        - nagios
    - name: Create nagiosadmin user account
      htpasswd:
        path: /usr/local/nagios/etc/htpasswd.users # required. Path to the file that contains the usernames and passwords
        name: nagiosadmin # required. User name to add or remove
        password: nagiosadmin # not required. Password associated with user.,Must be specified if user does not exist yet.
      tags:
        - nagios
    - name: start apache
      service:
        name: apache2
        state: started
        enabled: true
      tags:
        - nagios
    - name: start nagios
      service:
        name: nagios
        state: started
        enabled: true
      tags:
        - nagios
    - name: Download the plugin to temp folder
      get_url:
        dest: "{{ nagios_plugin_location }}"
        url: "https://github.com/nagios-plugins/nagios-plugins/archive/release-{{ nagios_plugin_version }}.tar.gz"
      tags:
        - plugin
    - name: untar the plugin code
      unarchive:
        src: "{{ nagios_plugin_location }}"
        dest: "/tmp/"
      tags:
        - plugin
    - name: Compile + Install
      shell:
        cmd: sudo ./tools/setup > setup.log && sudo ./configure > configue.log && sudo make > make.log && sudo make install
      args:
        chdir: "/tmp/nagios-plugins-release-{{ nagios_plugin_version }}/"
      tags:
        - plugin
    - name: restart nagios
      service:
        name: nagios
        state: restarted
      tags:
        - plugin
    - name: restart apache
      service:
        name: apache2
        state: restarted
      tags:
        - plugin
