# ansible-logstash

```
! This repository is in beta and does not contain content which can be used without adaptions so far
```

## what is ansible

Ansible is an IT automation tool. It can configure systems, deploy software, and orchestrate more advanced IT tasks and its free.
There is a ui which costs money.
Ansible executes command on remote hosts by using ssh.
Thats the idea and the rest of it is syntactic sugar and additional workarounds if native openSSH is not available in a fitting version.

http://www.ansible.com/home
http://docs.ansible.com/index.html

1. Install it:

http://docs.ansible.com/intro_installation.html

If the hosts can change, disbale host key checkeing e.g.g by

    $ export ANSIBLE_HOST_KEY_CHECKING=False

2.Create an inventory file:

Thats a file which contains known hosts, groups them together and add varaibles related to the hosts. Its stored in /etc/ansible/hosts or you have to add the file as parameter if you want to execute commands
A file could look like this:

    mail.example.com
    other2.example.com ansible_connection=ssh ansible_ssh_user=mdehaan

    [webservers]
    www[01:50].example.com

    [webservers:vars]
    ntp_server=ntp.atlanta.example.com
    proxy=proxy.atlanta.example.com

    [dbservers]
    one.example.com
    two.example.com
    
    [databases]
    db-[a:f].example.com
    
Details can be found at: http://docs.ansible.com/intro_inventory.html

3.Execute commands

e.g. hello on all hosts in /etc/ansible/hosts

    ansible all -a "/bin/echo hello"

e.g. hello on the host1 (limit) from the local host file (-i)
    
    ansible -i ./hosts --limit localhost all -a "/bin/echo hello"

4.Playbooks

Instead of executing ad-hoc commands, you probably want to write scripts.
They are called playbooks and they are written in yaml syntax.
You can use commands in this playbooks or you can use an so called ansible modules (http://docs.ansible.com/modules.html, http://docs.ansible.com/modules_by_category.html) which aggregate environment commands userfriendly.

Again a simple example (playbook.yml):

    ---
    - hosts: webservers
      vars:
        http_port: 80
        max_clients: 200
      remote_user: root
      tasks:
      - name: ensure apache is at the latest version
         yum: pkg=httpd state=latest
      - name: write the apache config file
         template: src=/srv/httpd.j2 dest=/etc/httpd.conf
        notify:
        - restart apache
      - name: ensure apache is running
        service: name=httpd state=started
      handlers:
         - name: restart apache
           service: name=httpd state=restarted

Each play contains a list of tasks. Tasks are executed in order, one at a time, against all machines matched by the host pattern, before moving on to the next task. Modules are ‘idempotent’, meaning if you run them again, they will make only the changes they must in order to bring the system to the desired state.

You can execute it (e.g. with parallelism level of 10)

    ansible-playbook playbook.yml -f 10

5.Handle more complex playbooks

The best way to handle more complex stuff is the usage of includes or so called roles.
See for example http://docs.ansible.com/playbooks_roles.html

Includes can look like this:

    - name: this is a play at the top level of a file
      hosts: all
      remote_user: root

      tasks:

      - name: say hi
        tags: foo
        shell: echo "hi..."

    - include: load_balancers.yml
    - include: webservers.yml
    - include: dbservers.yml

Roles are just automation around ‘include’ directives by conventions.
If you would organize your structure like this:

    site.yml
    webservers.yml
    fooservers.yml
    roles/
       common/
         files/
         templates/
         tasks/
         handlers/
         vars/
         meta/
       webservers/
         files/
         templates/
         tasks/
         handlers/
         vars/
         meta/

You could simple write something like this:

    ---
    - hosts: webservers
      roles:
         - common
         - webservers

6.Where can you find more:

- https://github.com/ansible/ansible-examples/tree/master/play-webapp
- https://galaxy.ansible.com


## What is logstash

logstash is a tool for managing events and logs. You can use it to collect logs, parse them, and store them for later use

Or in my words:
- It can use several inputs (e.g. logs, streams, ...)
- It can manipulate the data (e.g. filtering, extracting fields, ...)
- It can push data out

So its for example perfect to collect log files from distributed nodes, filter data and send the data to a cetral datastore where the data can be analyzed. 

### the setup on distributed nodes

In this example I will use logstash to parse local logfiles and push that data to a central redis database. This part is called shipper

On a central node (where the database is stored), I will use the data from the redis database to add this data to an elastic search database which then can be used by kibana to show the log data. I will call it indexer.

This setup can be used to handle several nodes, because it collect the data from the nodes and mergen the data together and because of the usage of redis as middleware it will perform well and is able to handle peaks.

### the indexer

I will assume we will use ubuntu and install everything in a folder called /monitoring

    sudo apt-get update
    sudo apt-get upgrade
    sudo mkdir /monitoring

We need java

    sudo apt-get install openjdk-6-jre-headless -y

We need redis

    sudo apt-get install redis-server -y
    cd /etc/redis
    sudo sed -i 's/bind 127.0.0.1/bind 0.0.0.0/g' redis.conf
    sudo service redis-server restart

We need elastic search with some nice plugins

    cd /monitoring
    sudo wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.0.1.tar.gz
    sudo tar zxvf elasticsearch-1.0.1.tar.gz
    sudo mv elasticsearch-1.0.1 elasticsearch
    cd /monitoring/elasticsearch/config/
    sudo mv elasticsearch.yml elasticsearch-Default.yml
    sudo sed 's/# cluster.name: elasticsearch/cluster.name: elasticsearch-monitoring/g'     elasticsearch-Default.yml > elasticsearch.yml
    cd /monitoring/elasticsearch/bin
    sudo ./plugin -install royrusso/elasticsearch-HQ
    sudo ./elasticsearch -d

And we need logstash itself

    cd /monitoring    
    sudo wget https://download.elasticsearch.org/logstash/logstash/logstash-1.4.0.tar.gz
    sudo tar zxvf logstash-1.4.0.tar.gz
    sudo  mv logstash-1.4.0 logstash
    cd /monitoring/logstash/bin
    
A config could look like [logstash-indexer.conf][2]    

    sudo ./logstash -f logstash-indexer.conf

### the shipper

What we need is only logstash e.g. by

    sudo wget https://download.elasticsearch.org/logstash/logstash/logstash-1.4.0.tar.gz
    sudo tar zxvf logstash-1.4.0.tar.gz
    
Then we should add the log files as input e.g. by a config see [logstashShipper.conf][1]

Then we can start the shipper by 

    sudo ./logstash -f logstashShipper.conf


## lets bring it together

TODO


[1]: https://github.com/michaelgruczel/ansible-logstash/blob/master/logstashShipper.conf
[2]: https://github.com/michaelgruczel/ansible-logstash/blob/master/logstash-indexer.conf
