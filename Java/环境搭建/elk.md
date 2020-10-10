##使用docker安装 elk （注意安装版本要对应）

###安装 elasticsearch

docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" registry.cn-hangzhou.aliyuncs.com/namespace_lyw/elasticsearch:7.9.0

###安装 logstash

####运行容器
    
    首先创建容器，将logstash的配置文件目录拷贝出来后删除容器
    docker run --name logstash -d -p 5044:5044 -p 9600:9600 registry.cn-hangzhou.aliyuncs.com/namespace_lyw/logstash:7.9.0
    docker cp logstash:/usr/share/logstash/config ~/settings/config
    
    修改logstash.yml 将xpack.monitoring.elasticsearch.hosts修改为elasticsearch对应ip+port
    修改pipeline.yml 将path.config修改成指定的logstash.conf路径
    创建logstash.conf 编辑以下为控制台输入，elasticsearch输出的案例
````   
input {
  stdin{}
}

output {
  elasticsearch {
    hosts => ["http://192.168.170.128:9200"]
    index => "applog"
    #user => "elastic"
    #password => "changeme"
  }
}
````   
    


###安装kibana

docker run --link YOUR_ELASTICSEARCH_CONTAINER_NAME_OR_ID:elasticsearch -p 5601:5601 registry.cn-hangzhou.aliyuncs.com/namespace_lyw/kibana:7.9.0





