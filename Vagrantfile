VAGRANTFILE_API_VERSION = "2"

$test = <<-SCRIPT
  nod_nam=$1
  nod_ip=$2
  is_first_node=$3

  echo $nod_nam
  echo $nod_ip
  echo $is_first_node

  #echo "Substitution with sed..."
  #sed -i 's/127.0.1.1/#127.0.1.1/g' /etc/hosts
  cat /vagrant/json/define_replicaset.json | mongo --host mongodb-configsvr1 --port 27017
  echo $?
SCRIPT

$install_keepalived_haproxy = <<-SCRIPT
  node=$1
  echo ""
  echo "add service and run haproxy and keepalived on srv"$node
  echo ""
  sudo apt install keepalived haproxy -y

  rm -fv /etc/haproxy/haproxy.conf
  rm -fv /etc/keepalived/keepalived.conf

  cp -pv /vagrant/srv${node}/haproxy.cfg /etc/haproxy/
  cp -pv /vagrant/srv${node}/keepalived.conf /etc/keepalived/
  chmod a-x /etc/keepalived/keepalived.conf

  #allow binding to virtual IP of keepalived and apply this adjustment
  echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
  sysctl -p

  systemctl daemon-reload
  systemctl start keepalived
  systemctl enable keepalived
  systemctl restart haproxy
  systemctl enable haproxy

SCRIPT

$install_misc = <<-SCRIPT
   apt update
   apt install mc -y
   echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n172.16.94.11       es-node1\n172.16.94.12       es-node2\n172.16.94.13       es-node3">> /etc/hosts
   sed -i 's/127.0.1.1/#127.0.1.1/g' /etc/hosts
SCRIPT

$install_es_cluster = <<-SCRIPT
   nod_nam=$1

   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
   echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
   sudo apt update
   sudo apt install apt-transport-https openjdk-8-jre-headless -y
   sudo apt update
   sudo apt install elasticsearch -y

   rm -fv /etc/elasticsearch/jvm.options
   rm -fv /etc/elasticsearch/elasticsearch.yml


   case $nod_nam in
     es-node1)
          echo "add service and run elasticsearch on"$nod_nam
          cp -v /vagrant/es-node1/jvm.options /etc/elasticsearch/jvm.options
          cp -v /vagrant/es-node1/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
          ;;
     es-node2)
          echo "add service and run elasticsearch on"$nod_nam
          cp -v /vagrant/es-node2/jvm.options /etc/elasticsearch/jvm.options
          cp -v /vagrant/es-node2/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml

          ;;
     es-node3)
          echo "add service and run elasticsearch on"$nod_nam
          cp -v /vagrant/es-node3/jvm.options /etc/elasticsearch/jvm.options
          cp -v /vagrant/es-node3/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
          ;;
     esac

     systemctl status elasticsearch
     systemctl start elasticsearch
SCRIPT


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "srv1" do |srv1|
    srv1.vm.box = "bento/ubuntu-18.04"
    srv1.vm.hostname = "srv1"
    srv1.vm.network :private_network, ip: "192.168.1.11"
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
       shell.inline = $install_keepalived_haproxy
       shell.args = ["1"]
    end
  end

  config.vm.define "srv2" do |srv2|
    srv2.vm.box = "bento/ubuntu-18.04"
    srv2.vm.hostname = "srv2"
    srv2.vm.network :private_network, ip: "192.168.1.12"
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
       shell.inline = $install_keepalived_haproxy
       shell.args = ["2"]
    end
  end

  config.vm.define "es1" do |es1|
    es1.vm.box = "bento/ubuntu-18.04"
    es1.vm.hostname = "es-node1"
    es1.vm.network :private_network, ip: "172.16.94.11"
	es1.vm.provider "virtualbox" do |esvm|
      esvm.memory = 1024
    end
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
        shell.inline = $install_es_cluster
        shell.args = ["es-node1"]
    end
  end

  config.vm.define "es2" do |es2|
    es2.vm.box = "bento/ubuntu-18.04"
    es2.vm.hostname = "es-node2"
    es2.vm.network :private_network, ip: "172.16.94.12"
	es2.vm.provider "virtualbox" do |esvm|
      esvm.memory = 1024
    end
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
        shell.inline = $install_es_cluster
        shell.args = ["es-node2"]
    end
  end

  config.vm.define "es3" do |es3|
    es3.vm.box = "bento/ubuntu-18.04"
    es3.vm.hostname = "es-node3"
    es3.vm.network :private_network, ip: "172.16.94.13"
	es3.vm.provider "virtualbox" do |esvm|
      esvm.memory = 1024
    end
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
        shell.inline = $install_es_cluster
        shell.args = ["es-node3"]
    end
   end


  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "512"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
