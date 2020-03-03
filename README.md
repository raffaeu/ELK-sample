# Introduction

This guide explains how to setup and run ELK on a Windows environment.

## System Requirements

In order to run ELK, the following components need to be downloaded

- JRE 1.8 version for Windows [JDK Downloads](https://www.oracle.com/java/technologies/javase-jre8-downloads.html)
- Elastic Search [ES Download Page](https://www.elastic.co/downloads/)
- Logstash [ES Download Page](https://www.elastic.co/downloads/)
- Kibana [ES Download Page](https://www.elastic.co/downloads/)
- Various Beats [ES Download Page](https://www.elastic.co/downloads/)

The installation is then split in two phases:

1. Install one or more servers where Elastic Search and Kibana will be hosted
2. Installa various Agents on the destination servers, such as beats and pipes

## Installation Process (Servers)

### 01. Install Elastic Search

The first step is to prepare Elastic Search server. Head to the Elastic search installation and open the configuration file ```elasticsearch.yml```, the following items must be modified:

```yaml
cluster.name: raf-monitor
node.attr.rack: raf-monitor-node-01

# the IP of the network adapter
network.host: 192.168.0.1
http.port: 9200

# needed when hosting on IP
discovery.seed_hosts: ["192.168.0.1"]
cluster.initial_master_nodes: ["raf-monitor-node-01"]
```

Then you can start the installation of the windows service by heading to the ```bin``` folder:

```powershell
PS C:\ELK-SETUP> cd .\Elastic\elasticsearch-7.6.0\bin\
PS C:\ELK-SETUP\Elastic\elasticsearch-7.6.0\bin>

# install the service
PS C:\ELK-SETUP\Elastic\elasticsearch-7.6.0\bin> .\elasticsearch-service.bat install
    Installing service      :  "elasticsearch-service-x64"

# start
PS C:\ELK-SETUP\Elastic\elasticsearch-7.6.0\bin> .\elasticsearch-service.bat start
      The service 'elasticsearch-service-x64' has been started
```

Now you should be able to browser Elastic Search on [http://localhost:9200](http://localhost:9200).

### 02. Install Logstash

 In order to install Logstash, we first need to modify the configuration to point correctly to our Elastic search server. First of all, rename the file ```logstash-sample.conf``` into ```syslog.conf``` and modify the following lines:

```yaml
output {
  elasticsearch {
    hosts => ["http://192.168.0.1:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

Second step, we can test Logstash and verify it is working and booting up correctly, via PShell:

```powershell
PS C:\ELK-SETUP\Logstash\logstash-7.6.0\bin>.\bin\logstash.bat -f .\config\syslog.conf
```

> Remember, if you do not pass the configuration file as a parameter with the ```-f```flag, Logstash will not boot correctly and throw an exception while starting up

> If you want, you can execute Logstash as a Windows service using products like NSSM and add a dependency to Elastic search so that on reboot it will ensure that also Elastic Search is up and running

### 03. Install Kibana

Kibana is the Dashboard provided by Elastic Serch team which allows to interact with Elastic Search and Logstash. The default port for Kibana is ```5601``` but you can open the configuration file ```kibana.yml``` and adjust it accordingly as shown below:

```yaml
server.port: 9800
elasticsearch.hosts: ["http://localhost:9200"]
#server.name: "your-hostname"
```

To run Kibana simply execute the ```kibana.bat``` file and wait that the web server boots correctly:

```powershell
PS C:\ELK-SETUP\Kibna\kibana-7.6.0\bin>.\bin\kibana.bat
```

> If you want, you can execute Kibana as a Windows service using products like NSSM and add a dependency to Elastic search so that on reboot it will ensure that also Elastic Search is up and running

## Install Process (Filebeat)

Everything is up and running but you still cannot read the logs provided by the various applications. In order to do so, there are two options:

1. Instruct your applications to send, via HTTP, Logstash logs over the network
2. Instruct your applications to write a stadard .log file on the filesystem and then install on each server a Filebeat which will parse the files and send them continuously to Logstash

You need to use option 2 because in case Logstash is down, Filebeat will keep checking for availability while option 1 will incur in let you loose some of your log files.

Open the file ```filebeat.yml``` and point correctly to your Elastic Search and Logstash server and inform filebeat where the files are stored:

```yaml
# Elastic search
output.elasticsearch:
  hosts: ["localhost:9200"]

# Where are the Logs

```