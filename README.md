# Confluent Kafka Monitoring with Zabbix

## Zabbix Server

### Enable Zabbix JMX Gateway

1. Install zabbix-java-gataway

   ``` bash
   yum install -y zabbix-java-gataway
   ```

2. Configuring zabbix-java-gataway

   ``` bash
   vi /etc/zabbix/zabbix_java_gateway.conf
   ```

   set `START_POLLERS=10`

3. Configuring zabbix-server

   ``` bash
   vi /etc/zabbix/zabbix_server.conf
   ```

   set to `StartJavaPollers=5` Change IP for `JavaGateway=IP_address_java_gateway`

4. restart zabbix-server and zabbix-java-gataway

   ``` bash
   systemctl restart zabbix-server
   systemctl start zabbix-java-gataway
   systemctl enable zabbix-java-gataway
   ```

### Install scripts for discovery JMX at Zabbix Server

``` bash
# cd /tmp/
# git clone https://github.com/peepeepopapapeepeepo/zabbix-kafka-monitoring.git
# cd zabbix-kafka-monitoring/zabbix-server/
# cp jmx_discovery /etc/zabbix/externalscripts
# cp JMXDiscovery-0.0.1.jar /etc/zabbix/externalscripts
# chmod +x /etc/zabbix/externalscripts/jmx_discovery
# chmod +x /etc/zabbix/externalscripts/JMXDiscovery-0.0.1.jar
```

## Kafka Each Servers

### Enable JMX Kafka Server

1. Create authentication files

   ``` bash
   # cd /etc/kafka/
   # _Username='kafka'
   # _Password='k@fKaZaB'
   # echo -e "${_Username}\t${_Password}" >> jmxremote.password
   # echo -e "${_Username}\treadonly" >> jmxremote.access
   # chown cp-kafka.confluent jmxremote.*
   # chmod 600 jmxremote.*
   ```

2. Edit systemd startup file

   ``` bash
   # systemctl edit confluent-kafka
        | [Service]
        |                  :                       :
       +| Environment="KAFKA_JMX_OPTS=-Dcom.sun.management.jmxremote=true \
       +| -Dcom.sun.management.jmxremote.ssl=false \
       +| -Djava.net.preferIPv4Stack=true \
       +| -Dcom.sun.management.jmxremote.local.only=false \
       +| -Dcom.sun.management.jmxremote.port=12345 \
       +| -Dcom.sun.management.jmxremote.rmi.port=12345 \
       +| -Djava.rmi.server.hostname=<HostIP> \
       +| -Dcom.sun.management.jmxremote.password.file=/etc/kafka/jmxremote.password \
       +| -Dcom.sun.management.jmxremote.access.file=/etc/kafka/jmxremote.access"
   # systemctl restart confluent-kafka
   ```

3. Test JMX

   ``` bash
   # mkdir /root/jmxterm && cd /root/jmxterm
   # curl -sLO https://github.com/jiaqi/jmxterm/releases/download/v1.0.1/jmxterm-1.0.1-uber.jar
   # java -jar jmxterm-1.0.1-uber.jar -l 127.0.0.1:12345 -u kafka -p k@fKaZaB
   $> beans
   $> info -b java.lang:type=Threading
   $> get -b kafka.server:name=PartitionCount,type=ReplicaManager Value
   $> get -b java.lang:type=Threading TotalStartedThreadCount
   $> exit
   # mkdir /root/jmxdump && cd /root/jmxdump
   # curl -sLO https://github.com/r4um/jmx-dump/releases/download/0.9.3/jmx-dump-0.9.3-standalone.jar
   # java -jar jmx-dump-0.9.3-standalone.jar -h 127.0.0.1 -p 12345 -c kafka:k@fKaZaB --dump-all
   ```

4. Allow firewalld

   ``` bash
   # firewall-cmd --permanent --new-ipset=jmx-zabbix --type=hash:ip
   # firewall-cmd --reload
   # firewall-cmd --permanent --ipset=jmx-zabbix --add-entry=<IP_address_java_gateway>
   # firewall-cmd --reload
   # firewall-cmd --permanent --new-service=jmx-zabbix
   # firewall-cmd --permanent --service=jmx-zabbix --set-description="JMX for Zabbix JMX Gateway"
   # firewall-cmd --permanent --service=jmx-zabbix --set-short="JMX for Zabbix"
   # firewall-cmd --permanent --service=jmx-zabbix --add-port=12345/tcp
   # firewall-cmd --reload
   # firewall-cmd --permanent --new-zone jmx-zabbix
   # firewall-cmd --reload
   # firewall-cmd --permanent --zone=jmx-zabbix --add-source=ipset:jmx-zabbix
   # firewall-cmd --permanent --zone=jmx-zabbix --add-service=jmx-zabbix
   # firewall-cmd --reload
   # firewall-cmd --info-ipset=jmx-zabbix
   # firewall-cmd --zone=jmx-zabbix --list-all
   # firewall-cmd --info-service=jmx-zabbix
   ```

### Install Burrow

1. Install Golang

   ``` bash
   # cd /tmp
   # curl -sLO https://storage.googleapis.com/golang/go1.13.linux-amd64.tar.gz
   # tar -C /usr/local -xzf /tmp/go1.13.linux-amd64.tar.gz
   # ln -s /usr/local/go/bin/go /usr/local/bin/go
   # ln -s /usr/local/go/bin/godoc /usr/local/bin/godoc
   # ln -s /usr/local/go/bin/gofmt /usr/local/bin/gofmt
   # export GOROOT=/usr/local/go
   # export PATH=$PATH:$GOROOT/bin
   # vi ~/.bash_profile
     +| export GOROOT=/usr/local/go
     +| export PATH=$PATH:$GOROOT/bin
   # go version
   ```

2. Compile and install Burrow

   ``` bash
   # mkdir -p /workspace/go/src/github.com/linkedin
   # cd /tmp
   # curl -sLO https://raw.githubusercontent.com/pote/gpm/v1.4.0/bin/gpm
   # chmod +x gpm
   # mv gpm /usr/local/bin/
   # yum install gcc
   # export GOPATH=/workspace/go
   # cd /workspace/go/src/github.com/linkedin/
   # git clone https://github.com/linkedin/Burrow.git
   # cd Burrow
   # go mod tidy
   # go install
   # ls -l $GOPATH/bin/Burrow
   ```

3. Configure Burrow

   ``` bash
   # mkdir -p /opt/burrow/bin/
   # cp $GOPATH/bin/Burrow /opt/burrow/bin/
   # cp -r config /opt/burrow/
   # cd /opt/burrow/config
   # cp burrow.toml burrow.toml.org
   ```

   edit `burrow.toml` to match your environment

   ``` bash
   # vi burrow.toml
       +| [general]
       +| pidfile="burrow.pid"
       +| stdout-logfile="burrow.out"
       +| access-control-allow-origin="*"
       +|  
       +| [logging]
       +| filename="logs/burrow.log"
       +| level="info"
       +| maxsize=100
       +| maxbackups=30
       +| maxage=10
       +| use-localtime=false
       +| use-compression=true
       +|  
       +| [zookeeper]
       +| servers=[ "kk1.kafka.local:2181", "kk2.kafka.local:2181", "kk3.kafka.local:2181" ]
       +| timeout=6
       +| root-path="/burrow"
       +|  
       +| [client-profile.test]
       +| client-id="burrow-test"
       +| kafka-version="0.10.0"
       +|  
       +| [cluster.local]
       +| class-name="kafka"
       +| servers=[ "kk1.kafka.local:9092", "kk2.kafka.local:9092", "kk3.kafka.local:9092" ]
       +| client-profile="test"
       +| topic-refresh=120
       +| offset-refresh=30
       +|  
       +| [consumer.local]
       +| class-name="kafka"
       +| cluster="local"
       +| servers=[ "kk1.kafka.local:9092", "kk2.kafka.local:9092", "kk3.kafka.local:9092" ]
       +| client-profile="test"
       +| group-blacklist="^(console-consumer-|python-kafka-consumer-|quick-).*$"
       +| group-whitelist=""
       +|  
       +| [consumer.local_zk]
       +| class-name="kafka_zk"
       +| cluster="local"
       +| servers=[ "kk1.kafka.local:2181", "kk2.kafka.local:2181", "kk3.kafka.local:2181" ]
       +| zookeeper-path="/kafka-cluster"
       +| zookeeper-timeout=30
       +| group-blacklist="^(console-consumer-|python-kafka-consumer-|quick-).*$"
       +| group-whitelist=""
       +|  
       +| [httpserver.default]
       +| address="127.0.0.1:8000"
       +|  
       +| [storage.default]
       +| class-name="inmemory"
       +| workers=20
       +| intervals=15
       +| expire-group=604800
       +| min-distance=1
   ```

4. Install startup script and start Burrow as linux service

   ``` bash
   # cd /tmp/
   # git clone https://github.com/peepeepopapapeepeepo/zabbix-kafka-monitoring.git
   # cd zabbix-kafka-monitoring/kafka-consumer/
   # cp burrow.service /etc/systemd/service/
   # systemctl daemon-reload
   # systemctl start burrow.service
   # systemctl enable burrow.service
   ```

5. Functional Test

   ``` bash
   # curl --noproxy "*" -s http://127.0.0.1:8000/v3/kafka/local/consumer | jq
   ```

### Configure Zabbix Agent

1. Install `kafka_consumers.sh`

   ``` bash
   # mkdir /var/lib/zabbix/
   # cd /tmp/zabbix-kafka-monitoring/kafka-consumer/
   # cp kafka_consumers.sh /var/lib/zabbix/
   # chmod +x /var/lib/zabbix/kafka_consumers.sh
   # chown -R zabbix.zabbix /var/lib/zabbix/
   ```

2. Install zabbix-agent configuration

   ``` bash
   # cp userparameter_kafkaconsumer.conf /etc/zabbix/zabbix_agentd.d
   # systemctl restart zabbix-agent
   ```

## Zabbix Web GUI

### Import template and value mapping file

1. Import template ([zbx_kafka_templates.xml](/kafka-server/zbx_templates_kafkaconsumers.xml), [zbx_templates_kafkaconsumers.xml](/kafka-consumer/zbx_templates_kafkaconsumers.xml))

   `Configuration` > `Templates` > `Import` > `Choose Files` > `Import`

2. Import value mapping ([zbx_valuemaps_kafkaconsumers.xml](/kafka-consumer/zbx_valuemaps_kafkaconsumers.xml))

   `Administration` > `General` > Drop Down List: `Value Mapping` > `Import` > `Choose Files` > `Import`

> **Note:** Don't forget to set macro `{$JMX_PASS}` and `{$JMX_USER}`