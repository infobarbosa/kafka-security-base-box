# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  #configuracao da instancia do zookeeper
  config.vm.define "zookeeper1" do |zookeeper1|
    zookeeper1.vm.box = "ubuntu/xenial64"

    zookeeper1.vm.network "private_network", ip: "192.168.56.11"
    zookeeper1.vm.hostname = "zookeeper1.infobarbosa.github.com"
    zookeeper1.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 1
      v.name = "zk1"
    end

    zookeeper1.vm.provision "file", source: "zookeeper.service", destination: "zookeeper.service"
    zookeeper1.vm.provision "file", source: "hosts", destination: "hosts"
    zookeeper1.vm.provision "shell", inline: <<-SHELL

      sudo add-apt-repository ppa:openjdk-r/ppa
      sudo apt-get -y update && sudo apt-get install -y openjdk-8-jdk
      wget -qO - https://packages.confluent.io/deb/5.0/archive.key | apt-key add -
      sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.0 stable main"
      sudo apt-get -y update && sudo apt-get install -y confluent-platform-2.11

      sudo ufw allow 2181/tcp

      sudo cat hosts >> /etc/hosts

      sudo cp zookeeper.service /etc/systemd/system/zookeeper.service
      chown root:root /etc/systemd/system/zookeeper.service
      sudo systemctl enable zookeeper
      sudo systemctl start zookeeper

    SHELL
  end

  #configuracao da instancia do kafka
  config.vm.define "kafka1" do |kafka1|
    kafka1.vm.box = "ubuntu/xenial64"

    kafka1.vm.network "private_network", ip: "192.168.56.12"
    kafka1.vm.hostname = "kafka1.infobarbosa.github.com"
    kafka1.vm.provider "virtualbox" do |v|
      v.memory = 1536
      v.cpus = 1
      v.name = "k1"
    end

    kafka1.vm.provision "file", source: "kafka.service", destination: "kafka.service"
    kafka1.vm.provision "file", source: "server1.properties", destination: "server.properties"
    kafka1.vm.provision "file", source: "hosts", destination: "hosts"
    kafka1.vm.provision "shell", inline: <<-SHELL
      sudo add-apt-repository ppa:openjdk-r/ppa
      sudo apt-get -y update && sudo apt-get install -y openjdk-8-jdk
      wget -qO - https://packages.confluent.io/deb/5.0/archive.key | apt-key add -
      sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.0 stable main"
      sudo apt-get -y update && sudo apt-get install -y confluent-platform-2.11

      sudo ufw allow 9092/tcp
      sudo ufw allow 9999/tcp

      sudo cat hosts >> /etc/hosts
      sudo mv server.properties /etc/kafka/server.properties

      sudo cp kafka.service /etc/systemd/system/kafka.service
      sudo chown root:root /etc/systemd/system/kafka.service
      sudo systemctl enable kafka
      sudo systemctl start kafka
    SHELL
  end

  #configuracao da instancia do control center
  config.vm.define "control-center" do |controlcenter|
    controlcenter.vm.box = "ubuntu/xenial64"

    controlcenter.vm.network "private_network", ip: "192.168.56.13"
    controlcenter.vm.hostname = "control-center.infobarbosa.github.com"
    controlcenter.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
      v.name = "confluent-control-center"
    end

    controlcenter.vm.provision "file", source: "control-center.service", destination: "control-center.service"
    controlcenter.vm.provision "file", source: "control-center.properties", destination: "control-center.properties"
    controlcenter.vm.provision "file", source: "hosts", destination: "hosts"
    controlcenter.vm.provision "shell", inline: <<-SHELL
      sudo add-apt-repository ppa:openjdk-r/ppa
      sudo apt-get -y update && sudo apt-get install -y openjdk-8-jdk
      wget -qO - https://packages.confluent.io/deb/5.0/archive.key | apt-key add -
      sudo add-apt-repository "deb [arch=amd64] https://packages.confluent.io/deb/5.0 stable main"
      sudo apt-get -y update && sudo apt-get install -y confluent-platform-2.11

      sudo ufw allow 9021/tcp

      sudo cat hosts >> /etc/hosts
      sudo mv control-center.properties /etc/confluent-control-center/control-center.properties

      sudo cp control-center.service /etc/systemd/system/control-center.service
      sudo chown root:root /etc/systemd/system/control-center.service
      sudo systemctl enable control-center
      sudo systemctl start control-center
    SHELL
  end

  #configuracao da instancia da aplicacao cliente
  config.vm.define "kafka-client" do |client|
    client.vm.box = "ubuntu/xenial64"

    client.vm.network "private_network", ip: "192.168.56.14"
    client.vm.hostname = "kafka-client.infobarbosa.github.com"
    client.vm.provider "virtualbox" do |v|
      v.memory = 512
      v.cpus = 1
      v.name = "kafka-client"
    end

    client.vm.provision "file", source: "hosts", destination: "hosts"
    client.vm.provision "file", source: "consumer.jar", destination: "consumer.jar"
    client.vm.provision "file", source: "producer.jar", destination: "producer.jar"
    client.vm.provision "shell", inline: <<-SHELL
      sudo add-apt-repository ppa:openjdk-r/ppa
      sudo apt-get -y update && sudo apt-get install -y openjdk-8-jdk
      sudo echo "BOOTSTRAP_SERVERS_CONFIG=kafka1:9092" >> /etc/profile

      sudo cat hosts >> /etc/hosts
    SHELL
  end

  #configuracao da instancia da autoridade certificadora (CA)
  config.vm.define "ca" do |ca|
    ca.vm.box = "ubuntu/xenial64"

    ca.vm.network "private_network", ip: "192.168.56.15"
    ca.vm.hostname = "ca.infobarbosa.github.com"
    ca.vm.provider "virtualbox" do |v|
      v.memory = 512
      v.cpus = 1
      v.name = "ca"
    end

    ca.vm.provision "file", source: "hosts", destination: "hosts"
    ca.vm.provision "shell", inline: <<-SHELL
      sudo add-apt-repository ppa:openjdk-r/ppa
      sudo apt-get -y update && sudo apt-get install -y openjdk-8-jdk

      sudo cat hosts >> /etc/hosts
    SHELL
  end

  #configuracao da instancia do kerberos
#  config.vm.define "kerberos" do |kerberos|
#    kerberos.vm.box = "centos/7"
#
#    kerberos.vm.network "private_network", ip: "192.168.56.16"
#    kerberos.vm.hostname = "kerberos.infobarbosa.github.com"
#    kerberos.vm.provider "virtualbox" do |v|
#      v.memory = 512
#      v.cpus = 1
#      v.name = "kerberos"
#    end
#
#    kerberos.vm.provision "file", source: "kdc.conf", destination: "kdc.conf"
#    kerberos.vm.provision "file", source: "kadm5.acl", destination: "kadm5.acl"
#    kerberos.vm.provision "file", source: "krb5.conf", destination: "krb5.conf"
#    kerberos.vm.provision "file", source: "hosts", destination: "hosts"
#    kerberos.vm.provision "shell", inline: <<-SHELL
#      sudo yum install -y krb5-server
#      sudo systemctl restart firewalld
#      sudo firewall-cmd --permanent --add-port=88/tcp
#      sudo firewall-cmd --reload
#      sudo cat hosts >> /etc/hosts
#
#      sudo chown root:root kdc.conf
#      sudo chown root:root kadm5.acl
#      sudo chown root:root krb5.conf
#      sudo chmod 644 kdc.conf
#      sudo chmod 644 kadm5.acl
#      sudo chmod 644 krb5.conf
#      sudo mv kdc.conf /var/kerberos/krb5kdc/
#      sudo mv kadm5.acl /var/kerberos/krb5kdc/
#      sudo mv krb5.conf /etc/
#
#      export REALM="KAFKA.SECURE"
#      export ADMINPW="this-is-unsecure"
#
#      sudo /usr/sbin/kdb5_util create -s -r KAFKA.SECURE -P this-is-unsecure
#      sudo kadmin.local -q "add_principal -pw this-is-unsecure admin/admin"
#
#      sudo systemctl restart krb5kdc
#      sudo systemctl restart kadmin
#
#    SHELL
#  end

end
