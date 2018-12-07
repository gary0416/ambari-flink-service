# ambari-flink-service

based on https://github.com/abajwa-hw/ambari-flink-service

升级flink版本至1.7.0, 并进行部分修改以适配HDP 2.6.3 (hadoop 2.7.3)

## flink编译
```
git clone https://github.com/apache/flink.git
cd flink
git checkout release-1.7.0
```
修改根的pom.xml
```
修改(避免org.apache.hadoop.yarn.client.RequestHedgingRMFailoverProxyProvider not found):
    hadoop.version为2.7.3.2.6.3.0-235
    zookeeper.version为3.4.6.2.6.3.0-235
增加(解决ClassNotFoundException: javax.ws.rs.ext.MessageBodyReader):
    <dependency>
        <groupId>javax.ws.rs</groupId>
        <artifactId>jsr311-api</artifactId>
        <version>1.1.1</version>
    </dependency>
```
找台ECS,配置阿里maven镜像(也可-Dhadoop.version=xx来修改版本)
```
mvn clean install -DskipTests -Pvendor-repos
```
编译等待1小时6分.

如果是maven3.3.x需要额外执行下面:
```
cd flink-dist 
mvn clean install -DskipTests -Pvendor-repos
cd ..
```
最后打包
```
cd flink-dist/target/flink-1.7.0-bin/
tar zcvf flink-1.7.0-bin-hadoop2.7.3.2.6.3.0-235-scala_2.11.tgz flink-1.7.0
```


## 安装
登录master机(即有/var/lib/ambari-server的机器)
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
git clone https://github.com/gary0416/ambari-flink-service.git /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK
(可选:修改configuration/flink-ambari-config.xml的flink_download_url到内网镜像或自己编译的版本)
ambari-server restart
```
在ambari界面上添加flink服务,have fun

## WordCount测试
```
su - flink
export HADOOP_CONF_DIR=/etc/hadoop/conf
/opt/flink/bin/flink run -m yarn-cluster examples/batch/WordCount.jar
```

## 注意
- 如果重复安装,需要删除agent机上的/tmp/flink.tgz,否则会认为已经安装过(见flink.py:44)

## 高可用
https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/jobmanager_high_availability.html#yarn-cluster-high-availability
- 将yarn的yarn.resourcemanager.am.max-attempts改成4(default 2 meaning a single JobManager failure is tolerated).
- ambari上,flink,flink_numcontainers从1改成2
- hdfs dfs -mkdir /apps/flink && hdfs dfs -chown flink:hadoop /apps/flink && hdfs dfs -chmod 755 /apps/flink
- yarn application中,点ApplicationMaster进入flink UI,点Job Manager可看到高可用已经是zk

## 附

### 原README

#### An Ambari Service for Flink
Ambari service for easily installing and managing Flink on HDP clusters.
Apache Flink is an open source platform for distributed stream and batch data processing
More details on Flink and how it is being used in the industry today available here: [http://flink-forward.org/?post_type=session](http://flink-forward.org/?post_type=session)


The Ambari service lets you easily install/compile Flink on HDP 2.5.3
- Features:
  - By default, downloads prebuilt package of Flink 1.2, but also gives option to build the latest Flink from source instead
  - Exposes flink-conf.yaml in Ambari UI 

Limitations:
  - This is not an officially supported service and *is not meant to be deployed in production systems*. It is only meant for testing demo/purposes
  - It does not support Ambari/HDP upgrade process and will cause upgrade problems if not removed prior to upgrade

Author: [Ali Bajwa](https://github.com/abajwa-hw)
- Thanks to [Davide Vergari](https://github.com/dvergari) for enhancing to run in clustered env
- Thanks to [Ben Harris](https://github.com/jamesbenharris) for updating libraries to work with HDP 2.5.3
#### Setup

- Download HDP 2.5 sandbox VM image (Sandbox_HDP_2.5_1_VMware.ova) from [Hortonworks website](http://hortonworks.com/products/hortonworks-sandbox/)
- Import Sandbox_HDP_2.3_1_VMware.ova into VMWare and set the VM memory size to 8GB
- Now start the VM
- After it boots up, find the IP address of the VM and add an entry into your machines hosts file. For example:
```
192.168.191.241 sandbox.hortonworks.com sandbox    
```
  - Note that you will need to replace the above with the IP for your own VM
  
- Connect to the VM via SSH (password hadoop)
```
ssh root@sandbox.hortonworks.com
```


- To download the Flink service folder, run below
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/abajwa-hw/ambari-flink-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK   
```

- Restart Ambari
```
#sandbox
service ambari restart

#non sandbox
sudo service ambari-server restart
```

- Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:

On bottom left -> Actions -> Add service -> check Flink server -> Next -> Next -> Change any config you like (e.g. install dir, memory sizes, num containers or values in flink-conf.yaml) -> Next -> Deploy

  - By default:
    - Container memory is 1024 MB
    - Job manager memory of 768 MB
    - Number of YARN container is 1
  
- On successful deployment you will see the Flink service as part of Ambari stack and will be able to start/stop the service from here:
![Image](../master/screenshots/Installed-service-stop.png?raw=true)

- You can see the parameters you configured under 'Configs' tab
![Image](../master/screenshots/Installed-service-config.png?raw=true)

- One benefit to wrapping the component in Ambari service is that you can now monitor/manage this service remotely via REST API
```
export SERVICE=FLINK
export PASSWORD=admin
export AMBARI_HOST=localhost

#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`


#get service status
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X GET http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#start service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#stop service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
```

- ...and also install via Blueprint. See example [here](https://github.com/abajwa-hw/ambari-workshops/blob/master/blueprints-demo-security.md) on how to deploy custom services via Blueprints

#### Use Flink

- Run word count job
```
su flink
export HADOOP_CONF_DIR=/etc/hadoop/conf
cd /opt/flink
./bin/flink run --jobmanager yarn-cluster -yn 1 -ytm 768 -yjm 768 ./examples/batch/WordCount.jar
```
- This should generate a series of word counts
![Image](../master/screenshots/Flink-wordcount.png?raw=true)

- Open the [YARN ResourceManager UI](http://sandbox.hortonworks.com:8088/cluster). Notice Flink is running on YARN
![Image](../master/screenshots/YARN-UI.png?raw=true)

- Click the ApplicationMaster link to access Flink webUI
![Image](../master/screenshots/Flink-UI-1.png?raw=true)

- Use the History tab to review details of the job that ran:
![Image](../master/screenshots/Flink-UI-2.png?raw=true)

- View metrics in the Task Manager tab:
![Image](../master/screenshots/Flink-UI-3.png?raw=true)

#### Other things to try

- [Apache Zeppelin](https://zeppelin.incubator.apache.org/) now also supports Flink. You can also install it via [Zeppelin Ambari service](https://github.com/hortonworks-gallery/ambari-zeppelin-service) for vizualization

More details on Flink and how it is being used in the industry today available here: [http://flink-forward.org/?post_type=session](http://flink-forward.org/?post_type=session)


#### Remove service

- To remove the Flink service: 
  - Stop the service via Ambari
  - Unregister the service
  
    ```
export SERVICE=FLINK
export PASSWORD=admin
export AMBARI_HOST=localhost
#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X DELETE http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#if above errors out, run below first to fully stop the service
#curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
    ```
   - Remove artifacts
    ```
    rm -rf /opt/flink*
    rm /tmp/flink.tgz
    ```   
