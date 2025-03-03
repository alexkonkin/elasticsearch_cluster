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
  echo    ""
  echo    "-----------------------------------------------------------"
  echo    "|     keepalived/haproxy has been selected to install     |"
  echo    "-----------------------------------------------------------"
  echo    ""

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

$configure_rsyslog = <<-SCRIPT
  echo    ""
  echo    "--------------------------------------------------------------------------"
  echo    "| rsyslog redirection to rsyslog server has been selected to install     |"
  echo    "--------------------------------------------------------------------------"
  echo    ""

  rm -fv /etc/rsyslog.d/50-default.conf
  #copy rsyslog file that transfers syslogs to rsyslog-server
  cp -v /vagrant/srv2/50-default.conf /etc/rsyslog.d/
  systemctl restart rsyslog
SCRIPT

$configure_filebeat = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|     filebeat has been selected to install     |"
  echo    "-------------------------------------------------"
  echo    ""
  #no ssl encryption to to some error (protocol mismatch 3, please see logstash part)
  wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
  apt-get install apt-transport-https -y
  echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
  apt-get update
  apt-get install filebeat
  cp -v /vagrant/srv2/filebeat/filebeat.yml /etc/filebeat/
  chown -v root:root /etc/filebeat/filebeat.yml
  chmod -v 0644 /etc/filebeat/filebeat.yml
  systemctl enable filebeat
  systemctl start filebeat
SCRIPT

$configure_node_exporter = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|  node_exporter has been selected to install   |"
  echo    "-------------------------------------------------"
  echo    ""
  export RELEASE="0.18.1"
  wget https://github.com/prometheus/node_exporter/releases/download/v${RELEASE}/node_exporter-${RELEASE}.linux-amd64.tar.gz
  tar xvf node_exporter-${RELEASE}.linux-amd64.tar.gz
  cd node_exporter-${RELEASE}.linux-amd64/
  cp -pv node_exporter /usr/sbin/
  useradd node_exporter -s /sbin/nologin
  mkdir -pv /etc/sysconfig/
  cp -pv /vagrant/node_exporter/node_exporter /etc/sysconfig/node_exporter
  chown -Rv node_exporter:node_exporter /etc/sysconfig/node_exporter
  cp -pv /vagrant/node_exporter/node_exporter.service /etc/systemd/system/node_exporter.service
  systemctl daemon-reload
  systemctl enable node_exporter
  systemctl start node_exporter
  systemctl status node_exporter
  echo    ""
  echo    "-----------------------------------------------------------------"
  echo    "|  don't forget to update prometheus.yml                        |"
  echo    "|  and restart prometheus server to add this exporter           |"
  echo    "-----------------------------------------------------------------"
  echo     "  - job_name: '"$(hostname)"'"
  echo     "      static_configs:"
  echo     "        - targets: ['"$(hostname -i)":9100']"
  echo    ""

SCRIPT

$configure_prometheus_es_exporter = <<-SCRIPT
  echo    ""
  echo    "-----------------------------------------------------------"
  echo    "| prometheus_es_exporter has been selected to install     |"
  echo    "-----------------------------------------------------------"
  echo    ""
  apt-get update
  apt install python3-pip -y
  pip3 install prometheus-es-exporter
  groupadd --system prometheus_es_exporter
  useradd -s /sbin/nologin -r -g prometheus_es_exporter prometheus_es_exporter
  mkdir -pv /etc/sysconfig/
  mkdir -pv /etc/prometheus-es-exporter/
  cp -v /vagrant/prometheus_es_exporter/prometheus_es_exporter /etc/sysconfig/
  cp -v /vagrant/prometheus_es_exporter/prometheus_es_exporter.service /etc/systemd/system/
  cp -v /vagrant/prometheus_es_exporter/exporter.cfg /etc/prometheus-es-exporter/exporter.cfg
  chown -Rv prometheus_es_exporter:prometheus_es_exporter /etc/sysconfig/prometheus_es_exporter
  systemctl daemon-reload
  systemctl enable prometheus_es_exporter
  systemctl start prometheus_es_exporter
SCRIPT

$install_grafana = <<-SCRIPT
  echo    ""
  echo    "-----------------------------------------------------------"
  echo    "|            grafana has been selected to install         |"
  echo    "-----------------------------------------------------------"
  echo    ""
  add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
  wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -
  apt-key list grafana
  apt-get update
  apt-get install grafana -y
  systemctl daemon-reload
  systemctl enable grafana-server
  systemctl start grafana-server
  systemctl status grafana-server
SCRIPT


$configure_prometheus = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|     prometheus has been selected to install   |"
  echo    "-------------------------------------------------"
  echo    ""

  export RELEASE="2.13.1"
  wget https://github.com/prometheus/prometheus/releases/download/v${RELEASE}/prometheus-${RELEASE}.linux-amd64.tar.gz
  tar xvf prometheus-${RELEASE}.linux-amd64.tar.gz
  cd prometheus-${RELEASE}.linux-amd64/
  groupadd --system prometheus
  grep prometheus /etc/group
  useradd -s /sbin/nologin -r -g prometheus prometheus
  mkdir -p /etc/prometheus/{rules,rules.d,files_sd}  /var/lib/prometheus
  cp prometheus promtool /usr/local/bin/
  cp -r consoles/ console_libraries/ /etc/prometheus/
  cp -rv consoles/ console_libraries/ /etc/prometheus/

  cp -pv /etc/systemd/system/prometheus.service /etc/systemd/system/
  cp -pv /vagrant/prometheus/prometheus.yml     /etc/prometheus/

  chown -Rv prometheus:prometheus /etc/prometheus/  /var/lib/prometheus/
  chmod -Rv 775 /etc/prometheus/ /var/lib/prometheus/
  promtool check config /etc/prometheus/prometheus.yml
  systemctl enable prometheus
  systemctl start prometheus
  systemctl status prometheus
SCRIPT



$install_misc = <<-SCRIPT
  echo    ""
  echo    "-------------------------------------------------"
  echo    "|   misc section has been selected to install   |"
  echo    "-------------------------------------------------"
  echo    ""

   apt update
   apt install mc -y
   echo -e "192.168.1.11       srv1\n192.168.1.12       srv2\n192.168.1.100       srv\n172.16.94.11       es-node1\n172.16.94.12       es-node2\n172.16.94.13       es-node3\n172.16.94.14       kibana\n172.16.94.15       logstash">> /etc/hosts
   sed -i 's/127.0.1.1/#127.0.1.1/g' /etc/hosts
SCRIPT

$install_es_cluster = <<-SCRIPT
   nod_nam=$1
   rsyslog_or_filebeat=$2

   wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
   echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
   sudo apt update
   sudo apt install apt-transport-https -y
   sudo apt update

   case $nod_nam in
     es-node1|es-node2|es-node3)
          echo "add service and run elasticsearch on"$nod_nam
          rm -fv /etc/elasticsearch/jvm.options
          rm -fv /etc/elasticsearch/elasticsearch.yml
          cp -v /vagrant/${nod_nam}/jvm.options /etc/elasticsearch/jvm.options
          cp -v /vagrant/${nod_nam}/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
          sudo apt install elasticsearch openjdk-8-jre-headless -y
          systemctl enable elasticsearch
          systemctl start elasticsearch
          systemctl status elasticsearch
          ;;
     kibana)
          echo "add service and run kibana on"$nod_nam
          sudo apt install kibana -y
          cp -pv /vagrant/kibana/kibana.yml /etc/kibana/
          systemctl enable kibana
          systemctl start kibana

          apt-get install nginx apache2-utils -y
          rm -fv /etc/nginx/sites-enabled/default
          cp -pv /vagrant/kibana/elk /etc/nginx/sites-available/
          ln -s /etc/nginx/sites-available/elk /etc/nginx/sites-enabled/elk
          htpasswd -b -c /etc/nginx/.elkusersecret admin admin
          systemctl restart nginx
          ;;
     logstash)
          echo "add service and run logstash on"$nod_nam
          #cp -v /vagrant/es-node3/jvm.options /etc/elasticsearch/jvm.options
          #cp -v /vagrant/es-node3/elasticsearch.yml /etc/elasticsearch/elasticsearch.yml
          apt install openjdk-8-jre-headless -y
          touch /etc/default/logstash
          apt install logstash -y
          systemctl stop logstash

          if [ $rsyslog_or_filebeat == "rsyslog" ]; then
            echo    ""
            echo    "------------------------------------"
            echo    "| rsyslog option nas been selected |"
            echo    "------------------------------------"
            echo    ""
            rm -fv /etc/logstash/conf.d/logstash.conf
            cp -v  /vagrant/logstash/logstash.conf /etc/logstash/conf.d/
            systemctl start logstash
            #configure rsyslog to accept data from remote hosts
            systemctl stop rsyslog
            rm -fv /etc/rsyslog.conf
            cp -v /vagrant/logstash/rsyslog.conf /etc/
            #validate configuration of central logging system
            rsyslogd -N1
            #copy json template for rsyslog
            cp -v /vagrant/logstash/01-json-template.conf /etc/rsyslog.d/01-json-template.conf
            #configure rsyslog to pass data to logstash
            cp -v /vagrant/logstash/60-output.conf /etc/rsyslog.d/60-output.conf
            systemctl restart rsyslog
            netstat -na | grep 10514
            curl -XGET 'http://srv:9200/_all/_search?q=*&pretty'
          fi
          if [ $rsyslog_or_filebeat == "filebeat" ]; then
            echo    ""
            echo    "------------------------------------"
            echo    "| filebeat option nas been selected |"
            echo    "------------------------------------"
            echo    ""
            #https://www.fosslinux.com/6084/how-to-install-elk-stack-on-ubuntu-18-04.htm
            #due to some problems ssl part is not working...config below skips this part
            cp -v /vagrant/logstash/filebeat/*.*  /etc/logstash/conf.d/
            systemctl enable logstash
            systemctl start logstash
          fi
          ;;
     esac
SCRIPT


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.define "srv1" do |srv1|
    srv1.vm.box = "bento/ubuntu-18.04"
    srv1.vm.hostname = "srv1"
    srv1.vm.network :private_network, ip: "192.168.1.11"
    config.vm.provision "misc", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "haproxy", type: "shell" do |shell|
       shell.inline = $install_keepalived_haproxy
       shell.args = ["1"]
    end
    config.vm.provision "node_exporter", type: "shell", run: "never" do |shell|
       shell.inline = $configure_node_exporter
    end
  end

  config.vm.define "srv2" do |srv2|
    srv2.vm.box = "bento/ubuntu-18.04"
    srv2.vm.hostname = "srv2"
    srv2.vm.network :private_network, ip: "192.168.1.12"
    config.vm.provision "misc", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "haproxy", type: "shell" do |shell|
       shell.inline = $install_keepalived_haproxy
       shell.args = ["2"]
    end
    config.vm.provision "logstash_rsyslog", type: "shell", run: "never" do |shell|
       shell.inline = $configure_rsyslog
    end
    config.vm.provision "logstash_filebeat", type: "shell", run: "never" do |shell|
       shell.inline = $configure_filebeat
    end
    config.vm.provision "node_exporter", type: "shell", run: "never" do |shell|
       shell.inline = $configure_node_exporter
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

  config.vm.define "kibana" do |kibana|
    kibana.vm.box = "bento/ubuntu-18.04"
    kibana.vm.hostname = "kibana"
    kibana.vm.network :private_network, ip: "172.16.94.14"
    kibana.vm.provider "virtualbox" do |esvm|
      esvm.memory = 1024
    end
    config.vm.provision "misc", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "es_cluster", type: "shell" do |shell|
        shell.inline = $install_es_cluster
        shell.args = ["kibana"]
    end
    config.vm.provision "prometheus", type: "shell", run: "never" do |shell|
        shell.inline = $configure_prometheus
    end
    config.vm.provision "prometheus_es_exporter", type: "shell", run: "never" do |shell|
        shell.inline = $configure_prometheus_es_exporter
    end
    config.vm.provision "grafana", type: "shell", run: "never" do |shell|
        shell.inline = $install_grafana
    end
   end

  config.vm.define "logstash" do |logstash|
    logstash.vm.box = "bento/ubuntu-18.04"
    logstash.vm.hostname = "logstash"
    logstash.vm.network :private_network, ip: "172.16.94.15"
    logstash.vm.provider "virtualbox" do |esvm|
      esvm.memory = 1024
    end
    config.vm.provision "script1", type: "shell" do |shell|
       shell.inline = $install_misc
    end
    config.vm.provision "script2", type: "shell" do |shell|
        shell.inline = $install_es_cluster
        shell.args = ["logstash","filebeat"]
    end
    config.vm.provision "node_exporter", type: "shell", run: "never" do |shell|
       shell.inline = $configure_node_exporter
    end
   end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "512"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end
