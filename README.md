## Automated ELK Stack Deployment

The files in this repository were used to configure the network depicted below.

https://drive.google.com/file/d/1VFzf9gedOQY9LRI5DyRi3IRrOdza4nUl/view?usp=sharing

These files have been tested and used to generate a live ELK deployment on Azure. They can be used to either recreate the entire deployment pictured above. Alternatively, select portions of the.yml file may be used to install only certain pieces of it, such as Filebeat.

# used to setup DVWA on all 3 web-servers
---                                                                                          - name: Install and setup DVWA dockers on Web-servers
  hosts: webservers
  become: true
  tasks:
  - name: install apache httpd 
    apt:
      name: apache2
      state: absent
  - name: Install docker.io
    apt:
      name: docker.io
      update_cache: yes
      state: present
  - name: Install Docker using pip
    pip:
      name: docker
      state: present
  - name: Install DVWA
    docker_container:
      name: dvwa
      image: cyberxsecurity/dvwa
      restart_policy: always
      published_ports: 80:80 #this is to allow http connection
      state: started
  - name: Enable docker service on restart
    systemd:
      name: docker
      enabled: yes

# used to setup elk server (KIBANA)
---
- name: Install and setup Elk docker on new ELK-VM
  hosts: elk
  become: true
  tasks:
  - name: Install docker.io
    apt:
      name: docker.io
      force_apt_get: yes
      update_cache: yes
      state: present
  - name: Install Python3-pip
    apt:
      force_apt_get: yes
      name: python3-pip
      state: present
  - name: Install Docker using pip
    pip:
      name: docker
      state: present
  - name: Increase Virtual Memory
    command: sysctl -w vm.max_map_count=262144
  - name: Use more Virtual Memory #to ensure it is persistent on reboot
    sysctl: 
      name: vm.max_map_count
      value: "262144"
      state: present
      reload: yes
  - name: Download and launch elk docker container
    docker_container:
      name: elk
      image: sebp/elk:761
      state: started
      restart_policy: always
      # these are the ports elk runs on
      published_ports:
        - 5601:5601
        - 9200:9200
        - 5044:5044

# Used to setup system logs from DVWA servers with filebeat on Kibana
---
- name: Install and setup Filebeat using our DVWA web-servers on Kibana
  hosts: webservers
  become: true
  tasks:
  - name: Download Filebeat.deb # ensure you are using latest ver/link
    command: curl -L -O https://artifacts.elastic.co/downloads/betas/filebeat/filebeat-7.6.1-amd64.deb
  - name: Install filebeat.deb # ensure you update the command for the latest version of filebeat
    command: dpkg -i filebeat-7.6.1-amd64.deb
  - name: copy to filebeat.yml
    copy:
      src: /etc/ansible/files/filebeat-config.yml
      dest: /etc/filebeat/filebeat.yml
  - name: enable and configure system module
    command: sudo filebeat modules enable system
  - name: setup filebeat # this can take a couple of minutes
    command: sudo filebeat setup
  - name: start filebeat service
    command: sudo service filebeat start

# Used to setup metric data from DVWA servers with metricbeat on Kibana
---
- name: Install and Launch Metricbeat using our DVWA web-servers on Kibana
  hosts: webservers
  become: true
  tasks:
  - name: Download metricbeat.deb # ensure you are using latest ver/link
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.6.1-amd64.deb
  - name: Install metricbeat.deb
    command: dpkg -i metricbeat-7.6.1-amd64.deb
  - name: Copy over and drop in metricbeat.yml
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml
  - name: enable and configure Docker module
    command: sudo metricbeat modules enable docker
  - name: setup metricbeat
    command: sudo metricbeat setup
  - name: start metricbeat
    command: sudo service metricbeat start

This document contains the following details:
- Description of the Topologu
- Access Policies
- ELK Configuration
  - Beats in Use
  - Machines Being Monitored
- How to Use the Ansible Build


### Description of the Topology

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the D*mn Vulnerable Web Application.

Load balancing ensures that the application will be highly Redundant, in addition to restricting DOS to the network.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the metric logs and system logs.

The configuration details of each machine may be found below.


| Name     |  Function  | IP Address |   Operating System   |
|----------|------------|------------|----------------------|
| Jump Box | Gateway    |  10.0.0.1  | Linux (ubuntu 20.04) |
| Web-1    | Redundancy |  10.0.0.8  | Linux (ubuntu 20.04) |
| Web-2    | Redundancy |  10.0.0.9  | Linux (ubuntu 20.04) |
| Web-3    | Redundancy |  10.0.0.11 | Linux (ubuntu 20.04) |   
### Access Policies

The machines on the internal network are not exposed to the public Internet. 

Only the Jump Box machine can accept connections from the Internet. Access to this machine is only allowed from the following IP addresses:
- 142.129.26.196

Machines within the network can only be accessed by the Jump Box's ansible docker container.


A summary of the access policies in place can be found in the table below.

|           Name              | Publicly Accessible | Allowed IP Addresses |
|-----------------------------|---------------------|----------------------|
| Jump Box                    | No                  | 142.129.26.196       |
| epic_nobel docker container | No                  |                      |
| Elk-vm                      | No                  | 142.129.26.196       |

### Elk Configuration

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because...
- Ansible reads YAML code which is not markup language and is designed to be very readable and easy to write

The playbook implements the following tasks:
- Install docker.io
- Install Python3-pip
- Install Docker using pip
- Download and launch elk docker container
- And more configurability seen above 

The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.

(C:\Users\ldani\OneDrive\Desktop\elkdocker.png)

### Target Machines & Beats
This ELK server is configured to monitor the following machines:
- Webserver 1 [10.0.0.9]
- Webserver 2 [10.0.0.8]
- Webserver 3 [10.0.0.11]

We have installed the following Beats on these machines:
- Webserver 1
- Webserver 2
- Webserver 3

These Beats allow us to collect the following information from each machine:
- `audit logs` collects accessed resources, login information, and other events
- `deprecation logs` uses elasticsearch to determine if certain functionality needs to be migrated
- `gc logs` garbage collection logging to monitor previous incoming requests
- `server logs` similar to audit logging, but more broad and can monitor client http requests, IP addresses, date and time, user agnts, and etc.
- `slow logs` help determine whether applications are correctly communicating with elastic search. Also examine queries to validate. Maintains cluster of logs

### Using the Playbook
In order to use the playbook, you will need to have an Ansible control node already configured. Assuming you have such a control node provisioned: 

SSH into the control node and follow the steps below:
- Copy the .yml playbook files to [/etc/filebeat , /etc/metricbeat]. # use the appropriate destination
- Update the .yml file to include webservers or elk hosts
- Run the playbook, and navigate to either webservers or elk vm to check that the installation worked as expected.

# When setting up DVWA, make sure to use webservers and SSH into either Web VMS to validate installation. then try to connect to each one using the public IP (52.250.13.172). If done properly, you should be in the login page for DVWA.

# When setting up the Elk server, make sure to use elk and SSH into the ELK-vm to validate the docker container is running. Try connecting to the ELK-VM public IP and port (http://104.42.215.204:5601/app/kibana). If done properly, kibana should load up and you should be in the kibana dashboard 
(C:\Users\ldani\OneDrive\Desktop\kibanadashboard.png)

# When setting up Filbeat on all 3 web-servers to communicate with the Elk Server, ensure you have a virtual network that can communicate both ways from elk to webservers. Then, in the .yml file  change hosts to webservers. If done properly, check in http://104.42.215.204:5601/app/kibana#/home/tutorial/systemLogs and scroll down and hit "check data". It should verify that there is data succesfully being received. 
(C:\Users\ldani\OneDrive\Desktop\systemlogssetup.png)

# When setting up Metricbeat on all 3 web-servvers, use the .yml file provided and ensure hosts is "webservers". If done properly, check in http://104.42.215.204:5601/app/kibana#/home/tutorial/dockerMetrics and scroll down and hit "check data." it should verify that there is data successfully being received.
(C:\Users\ldani\OneDrive\Desktop\metricbeatsetup.png)

Commands that should be used when utilizing playbooks
- ansible-playbook [filename].yml
- to edit configs, use nano or vi [filename.yml]
