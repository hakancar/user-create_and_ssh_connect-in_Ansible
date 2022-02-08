# user-create_and_ssh_connect-in_Ansible

Creating users and groups, adding users to the groups 
and authorising users to the servers for ssh connection with Ansible

Part-1 Setup Ansible
Part-2 Creating Users and Groups on Ubuntu (or Amazon Linux, what you have)
Part-2.a Creating Users and Groups
Part-2.b Adding users to the groups

Part-3 Adding User public keys to the authorised_key file


Part-1 Setup Ansible

Part-1.a Creating Servers

Create VMs in the cloud as you like. We will have 3 ubuntu, 1 amazon linux in this task.
After you created the EC2s, select controler and the nodes.
Connect to the controller
# ssh connect to the controller
ssh -i <~/.ssh/your_pem_file> <user-name>@<public-ip>

# ansible setup to the controller
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install ansible -y
ansible --version
You will see as below:
# ubuntu@ip-172-31-91-37:~$ ansible --version
# ansible 2.9.6
#   config file = /etc/ansible/ansible.cfg
#   configured module search path = ['/home/ubuntu/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
#   ansible python module location = /usr/lib/python3/dist-packages/ansible
#   executable location = /usr/bin/ansible
#   python version = 3.8.10 (default, Nov 26 2021, 20:14:08) [GCC 9.3.0]
We saw that we setup the Ansible correctly

Part-1.b Editing Host file, Pinging to the Nodes and some simple commands

cd /etc/ansible/
edit the hosts file in ansible directory

[ubuntuservers]
ubuntu-1 ansible_host=172.31.80.217 ansible_user=ubuntu
ubuntu-2 ansible_host=172.31.85.205 ansible_user=ubuntu

[linuxservers]
linux ansible_host=172.31.16.23 ansible_user=ec2-user

[all:vars]
ansible_ssh_private_key_file=/home/ubuntu/xxx.pem

Copy the pem file from location where you have the pem file to the controller server
scp -i xxx.pem xxx.pem ubuntu@107.22.101.45:/home/ubuntu

Change mod of the pem file
chmod 400 xxx.pem

Control the hosts by below command
ansible all --list-hosts

Ansible ping to the servers with ad-hoc command
ansible -m ping all

You can see the result shorter with -o
ansible all -m ping -o
ubuntu-1 | SUCCESS => {"ansible_facts": {"discovered_interpreter_python": 
"/usr/bin/python3"},"changed": false,"ping": "pong"}

Create a testfile in controller node to ubuntuservers. 
echo hello > testfile.txt

This command does not run for linux node
ansible all -m copy -a "src=/home/ubuntu/testfile.txt dest=/home/ubuntu/testfile.txt"

See the content of that file
ansible ubuntu-1 -m shell -a "cat testfile.txt"
ubuntu-1 | CHANGED | rc=0 >>
hello

Create a new file and see the error for linux node
ansible all -m shell -a "echo this is new file > /home/ubuntu/newfile"
linux | FAILED | rc=1 >>
/bin/sh: /home/ubuntu/newfile: No such file or directorynon-zero return code
ubuntu-1 | CHANGED | rc=0 >>
ubuntu-2 | CHANGED | rc=0 >>

See the content of that file for ubuntu nodes
ansible all -m shell -a "echo this is new file > /home/ubuntu/newfile ; cat newfile"
ubuntu-1 | CHANGED | rc=0 >>
this is new file
ubuntu-2 | CHANGED | rc=0 >>
this is new file
linux | FAILED | rc=1 >>
/bin/sh: /home/ubuntu/newfile: No such file or directory
cat: newfile: No such file or directorynon-zero return code

Run that code for linux node and see the content
ansible linux  -m shell -a "echo this is new file > /home/ec2-user/newfile ; cat newfile"
linux | CHANGED | rc=0 >>
this is new file

****************************************************

Ansible ping to the servers with play-book
create ping.yml file as below
---
- name : ping all servers
  hosts: all
  tasks:
   - name: ping test
     ping:

and then enter the code as below
ansible-playbook ping.yml


Part-2 Creating Users and Groups

Part-2.a Manual creating users as users.yml

nano users.yml
---
- name: creating users
  hosts: "*" #all is possible
  tasks:
   - user: 
      name: name1
      state: present
      name: name2
      state: present

Run that command ansible-playbook -b users.yml #mind -b for sudo
See the users in the servers in the /etc/passwd file with cat command
Change state= absent and see the result in the servers. You will see the user will be deleted

Part-2.b Creating users with "item"

---
- name: creating users with loop
  hosts: all
  become: true
  tasks:
   - user:
      name: "{{ item }}"
      state: present
     loop:
      - name1
      - name2
      - name3
      - name4
    
Again run that command ansible-playbook users.items.yml
See the users in the servers in the /etc/passwd file with cat command

Part-2.c Creating groups

---
- name: creating groups with loop
  hosts: all
  become: true
  tasks:
   - group:
      name: "{{ item }}"
      state: present
     loop:
      - group1
      - group2

run that command ansible-playbook groups.yml 
and see the users in the servers in the /etc/group file with cat command


Part-3 Adding User public keys to the authorised_key file

---
- name: Adding User public keys to the authorised_key file
  hosts: all
  tasks:
    - authorized_key:
       user: name1
       state: present
       key: "{{ lookup('file', './user_id_rsa.pub') }}"



      ssh -i id_rsa name1@public-ip-of-the-server #name1 is important. Because we are coming from name1 user


Finally, all script is in a block:

---
- name: creating users and groups and ssh connection
  hosts: "*" #all is possible
  become: yes
  tasks:
   - user:
      name: "{{ item }}"
      state: present
     loop:
      - name1
      - name2
      - name3
      - name4     
   - group:
      name: "{{ item }}"
      state: present
     loop:
      - group1
      - group2

   - ansible.builtin.user:
      name: name1
      shell: /bin/bash
      groups: group1,group2
      append: yes
      
   - ansible.builtin.user:
      name: name2
      shell: /bin/bash
      groups: group2
      append: yes

   - authorized_key:
       user: name1
       state: present
       key: "{{ lookup('file', './user_id_rsa.pub') }}"

   - authorized_key:
       user: name2
       state: present
       key: "{{ lookup('file', './user_id_rsa.pub') }}"
       
// if you need the same process for other users, you can add this script to the main block
   # - authorized_key:
   #     user: name3
   #     state: present
   #     key: "{{ lookup('file', './user_id_rsa.pub') }}"
