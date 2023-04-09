# ecs-fluentbit

## ECS Log Routing by source

application container 에서 raw format log 와 json format log 로 생성하고 firelens log driver 를 이용하여 cloudwatch 의 서로 다른 loggroup 으로 라우팅 

### Nginx 설정
nginx/nginx.conf 파일 참조

'''
#### raw format 로깅하기 위한 설정
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

access_log  /var/log/nginx/access_main.log  main;

#### json format 로깅하기 위한 설정
log_format json escape=json
    '{'
        '"time_local":"$time_local",'
        '"remote_addr":"$remote_addr",'
        '"remote_user":"$remote_user",'
        '"request":"$request",'
        '"status": "$status",'
        '"body_bytes_sent":"$body_bytes_sent",'
        '"request_time":"$request_time",'
        '"http_referrer":"$http_referer",'
        '"http_user_agent":"$http_user_agent"'
    '}';

access_log  /var/log/nginx/access_json.log  json;
'''     
          
        
### Fluentbit 설정 (firelens.conf)
'''
[FILTER]
    Name                grep
    Match               *-firelens-*
    Exclude             $log ^(?=.*ELB-HealthChecker\/2\.0).*$


[FILTER]
    Name                rewrite_tag
    Match               *-firelens-*
    Rule                $log "^[^{]" main-log-$container_id false

[FILTER]
    Name                rewrite_tag
    Match               *-firelens-*
    Rule                $log "^{" json-log-$container_id false

[OUTPUT]
    Name                cloudwatch_logs
    Match               main-log*
    region              ap-northeast-1
    log_group_name      /ecs/main-log
    log_stream_prefix   fluentbit-
    auto_create_group   true
    log_retention_days  30

[OUTPUT]
    Name                cloudwatch_logs
    Match               json-log*
    region              ap-northeast-1
    log_group_name      /ecs/json-log
    log_stream_prefix   fluentbit-
    auto_create_group   true
    log_retention_days  30
'''

### Custom Fluentbit Config Dockerfile
'''
FROM public.ecr.aws/aws-observability/aws-for-fluent-bit
COPY firelens.conf /firelens.conf
'''

### Nginx Dockerfile
'''
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
RUN ln -sf /dev/stdout /var/log/nginx/access_main.log && ln -sf /dev/stdout /var/log/nginx/access_json.log
'''

### ECS Task 결과 확인

/ecs/main-log cloudwatch log
'''
{
  "log": "27.0.3.153 - - [09/Apr/2023:12:44:35 +0000] \"GET / HTTP/1.1\" 304 0 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36\" \"-\"",
  "container_id": "4ff854a23a4b569af2c07066826711a9d5b3041d5857dad147ac7d80a7371b99",
  "container_name": "/ecs-log-seperate-to-cloudwatch-3-app-d8a2b9c1af99abc06f00",
  "source": "stdout",
  "ec2_instance_id": "i-021731e25b34189da",
  "ecs_cluster": "test-cluster",
  "ecs_task_arn": "arn:aws:ecs:ap-northeast-1:account-id:task/test-cluster/6cf60188b5e34ab49b98ac9e3a74bb9f",
  "ecs_task_definition": "log-seperate-to-cloudwatch:3"
}
'''

/ecs/json-log cloudwatch log
'''
{
  "source": "stdout",
  "log": "{
    \"time_local\":\"09/Apr/2023:12:59:12 +0000\",
    \"remote_addr\":\"185.225.74.198\",
    \"remote_user\":\"\",
    \"request\":\"GET /.git/config HTTP/1.1\",
    \"status\": \"404\",
    \"body_bytes_sent\":\"153\",
    \"request_time\":\"0.000\",
    \"http_referrer\":\"\",
    \"http_user_agent\":\"Mozilla/5.0 (Linux; U; Android 4.4.2; en-US; HM NOTE 1W Build/KOT49H) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 UCBrowser/11.0.5.850 U3/0.8.0 Mobile Safari/534.30\"
  }",
  "container_id": "4ff854a23a4b569af2c07066826711a9d5b3041d5857dad147ac7d80a7371b99",
  "container_name": "/ecs-log-seperate-to-cloudwatch-3-app-d8a2b9c1af99abc06f00",
  "ec2_instance_id": "i-021731e25b34189da",
  "ecs_cluster": "test-cluster",
  "ecs_task_arn": "arn:aws:ecs:ap-northeast-1:account-id:task/test-cluster/6cf60188b5e34ab49b98ac9e3a74bb9f",
  "ecs_task_definition": "log-seperate-to-cloudwatch:3"
}
'''