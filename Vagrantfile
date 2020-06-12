# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define "archive-elk", primary: true, autostart: true do |ps|
    # set box to official CentOS 7 image
    ps.vm.box = "centos/7"
    # explcitly set shared folder to virtualbox type. If not set will choose rsync
    # which is just a one-way share that is less useful in this context
    ps.vm.synced_folder "..", "/vagrant", type: "virtualbox"
    # Set hostname
    ps.vm.hostname = "archive-elk"

    #increase memory
    ps.vm.provider "virtualbox" do |v|
        v.memory = 4096
    end

    # Enable IPv4. Cannot be directly before or after line that sets IPv6 address. Looks
    # to be a strange bug where IPv6 and IPv4 mixed-up by vagrant otherwise and one
    #interface will appear not to have an address. If you look at network-scripts file
    # you will see a mangled result where IPv4 is set for IPv6 or vice versa
    ps.vm.network "private_network", ip: "10.7.7.7"

    # Setup port forwarding to apache
    ps.vm.network "forwarded_port", guest: 15672, host: "15672", host_ip: "198.111.224.158"
    ps.vm.network "forwarded_port", guest: 9200, host: "9200", host_ip: "198.111.224.158"
    ps.vm.network "forwarded_port", guest:5601, host: "5601", host_ip: "198.111.224.158"
    # Enable IPv6. Currently only supports setting via static IP. Address below in the
    # reserved local address range for IPv6
    ps.vm.network "private_network", ip: "fdac:218a:75e5:79c8::7"

    #Disable selinux
    ps.vm.provision "shell", inline: <<-SHELL
        sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config
    SHELL

    #Install all requirements and perform initial setup
    ps.vm.provision "shell", inline: <<-SHELL
        yum install -y epel-release yum-utils
        #need erlang for rabbitmq
        yum install -y http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm
        rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
        cp /vagrant/toolkit/files/elastic.repo /etc/yum.repos.d/elastic.repo
        cp /vagrant/central-archive/files/rabbit.repo /etc/yum.repos.d/rabbit.repo
        yum clean all
        #Install a few things first including java for logstash
        yum install -y \
            gcc\
            kernel-devel\
            kernel-headers\
            dkms\
            make\
            bzip2\
            python3 \
            python3-requests\
            elasticsearch\
            erlang\
            rabbitmq-server\
            java-11-openjdk

        # Esmond installs an old java for cassandra. Use the newer version from
        # Elastic for logstash. Interstingly ElasticSearch ships with its own version
        # of Java 13 that lives in /usr/share/elasticsearch/jdk
        cp /vagrant/toolkit/files/java.sh /etc/profile.d/java.sh
        source /etc/profile.d/java.sh

        #Use Java 11 or jruby script that setsup systemd scripts fails
        JAVA_HOME=/usr/lib/jvm/jre-11 yum install -y logstash

        #Logstash needs Java 11 to install, but 13 that comes with elastic seems fine after that
        echo "JAVA_HOME=/usr/share/elasticsearch/jdk" >> /etc/sysconfig/logstash

        #daemon reload
        systemctl daemon-reload

        #switch to shared directory
        cd /vagrant

        #CONFIGURE RABBITMQ
        systemctl enable --now rabbitmq-server
        rabbitmq-plugins enable rabbitmq_management

        #CONFIGURE ELASTIC
        systemctl enable --now elasticsearch.service
        /vagrant/toolkit/scripts/pselastic_secure.sh
        #user setup for rabbitmq
        rabbitmqctl add_user "elastic" "elastic"
        rabbitmqctl set_permissions -p "/" "elastic" ".*" ".*" ".*"
        #CONFIGURE LOGSTASH
        cp -r /vagrant/logstash_pipeline/* /etc/logstash/conf.d/
        cp -f /vagrant/toolkit/files/99-outputs.conf /etc/logstash/conf.d/99-outputs.conf
        ## Probably a better way to do this, but symlink to match directory structure of docker
        ln -s /etc/logstash/conf.d /usr/share/logstash/pipeline
        #Enable and start logstash
        systemctl enable --now logstash.service
        #kibana setup
        #rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
        #wget https://artifacts.elastic.co/downloads/kibana/kibana-7.7.0-x86_64.rpm
        #sudo yum install kibana
        #sudo /bin/systemctl daemon-reload
        #sudo /bin/systemctl enable kibana.service
        #sudo echo "service.host: \"0.0.0.0\"" >> /etc/kibana/kibana.yml
        #password swap sudo sed -i 's/elasticsearch.password\:.*/elasticsearch.password:""/' /etc/kibana/kibana.yml
        #turn off encryption at /etc/perfsonar/elastic/auth_setup
        #reload elasticsearch
        #sudo systemctl start kibana.service
    SHELL
  end
  config.vm.define "testpoint1" do |testpoint1|
        #testpoint1.vm.box = "debian/stretch64"
        testpoint1.vm.box = "geerlingguy/ubuntu1604"
        #testpoint1.vm.provision "shell", inline: <<-SHELL
                #cd /etc/apt/sources.list.d/
                #sudo wget http://downloads.perfsonar.net/debian/perfsonar-release.list
                #cd
                #sudo wget -qO - http://downloads.perfsonar.net/debian/perfsonar-official.gpg.key | apt-key add -
                #echo "10.7.7.7 archive-elk" >> /etc/hosts
                #sudo apt-get update
                #sudo apt-get --yes install perfsonar-testpoint
        #SHELL
   end

end
