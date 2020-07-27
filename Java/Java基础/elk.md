##使用docker安装 elk （注意安装版本要对应）

###安装 elasticsearch

docker run -di -p 9200:9200 --name elasticsearch docker.elastic.co/elasticsearch/elasticsearch:6.6.2 

###安装 logstash

####运行容器

docker run -di -p 5044:5044 --name logstash docker.elastic.co/logstash/logstash:6.6.2

####修改conf/logstash.yml 将url修改为 ip:port

####修改pipeline/logstash.conf

- 指定管道输入与输出
- 在与springboot集成时 需要指定tcp方式的管道输入
 eg: input { tcp { port => 5044 codec => json_lines} }

-指定elasticsearch 来做logstash的管道输出 将日志发送到elasticsearch中 (index：指定一个索引,hosts：指定elasticsearch的主机端口)
eg: output { elasticsearch { hosts => "192.168.50.154:9200" index => "applog"} stdout { } }

此命令用来测试 logstash和elasticsearch 是否可以通讯

docker run -it --rm docker.elastic.co/logstash/logstash:6.6.2 -e 'input { stdin { } tcp { port => 5044 codec => json_lines} } output { elasticsearch { hosts => "192.168.50.154:9200" index => "applog" } stdout { } }'


###安装kibana
docker run --name kibana -e ELASTICSEARCH_URL=http://192.168.50.154:9200 -p 5601:5601 -d docker.elastic.co/kibana/kibana:6.6.2


###springboot集成 elk
- 导logback的jar包
- 编写logback.xml（注意 此配置文件比application.yml优先级更高 所以application.name需要写在此配置文件中）


