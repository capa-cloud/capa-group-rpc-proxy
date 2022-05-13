## Nginx运维监控

### status监控界面

#### 开启status

### 日志

默认访问日志格式：

```text
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for" $request_time $upstream_response_time';
```

默认格式日志样例：

```text
39.105.66.117 - mp [11/Sep/2019:19:03:01 +0800] "POST /salesplatform-gateway/users HTTP/1.1" 200 575 "-" "Apache-HttpClient/4.5.5 (Java/1.8.0_161)" "-" 0.040 0.040

39.105.66.117 - mp [11/Sep/2019:19:03:08 +0800] "POST /salesplatform-gateway/users HTTP/1.1" 200 575 "-" "Apache-HttpClient/4.5.5 (Java/1.8.0_161)" "-" 0.008 0.008
```

* $remote_addr: 客户端的ip地址
* $remote_user: 用于记录远程客户端的用户名称
* $time_local: 用于记录访问时间和时区
* $request: 用于记录请求的url以及请求方法
* $status: 响应状态码
* $body_bytes_sent: 给客户端发送的文件主体内容字节数
* $http_referer: 可以记录用户是从哪个链接访问过来的
* $http_user_agent: 用户所使用的浏览器信息
* $http_x_forwarded_for: 可以记录客户端IP，通过代理服务器来记录客户端的ip地址
* $request_time: 指的是从接受用户请求的第一个字节到发送完响应数据的时间，即\$request_time包括接收客户端请求数据的时间、后端程序响应的时间、发送响应数据给客户端的时间
* $upstream_response_time: 用于接收来自上游服务器的响应的时间

#### 常用分析命令

1、根据访问IP统计UV

    awk '{print $1}' paycenteraccess.log | sort -n | uniq | wc -l

2、查询访问最频繁的IP(前10)

    awk '{print $1}' /var/log/nginx/access.log | sort -n |uniq -c | sort -rn | head -n 10

3、查看某一时间段的IP访问量(1-8点)

    awk '$4 >="[25/Mar/2020:01:00:00" && $4 <="[25/Mar/2020:08:00:00"' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c| sort -nr |wc -l

4、查看访问100次以上的IP

    awk '{print $1}' /var/log/nginx/access.log | sort -n |uniq -c |awk '{if($1 >100) print $0}'|sort -rn

5、查看指定ip访问过的url和访问次数

    grep "39.105.67.140" /var/log/nginx/access.log|awk '{print $7}' |sort |uniq -c |sort -n -k 1 -r

6、根据访问URL统计PV

    cat /var/log/nginx/access.log |awk '{print $7}' |wc -l

7、查询访问最频繁的URL(前10)

    awk '{print $7}' /var/log/nginx/access.log | sort |uniq -c | sort -rn | head -n 10

8、查看访问最频的URL([排除/api/appid])(前10)

    grep -v '/api/appid' /var/log/nginx/access.log|awk '{print $7}' | sort |uniq -c | sort -rn | head -n 10

9、查看页面访问次数超过100次的页面

    cat /var/log/nginx/access.log | cut -d ' ' -f 7 | sort |uniq -c | awk '{if ($1 > 100) print $0}' | less

10、查看最近1000条记录，访问量最高的页面

    tail -1000 /var/log/nginx/access.log |awk '{print $7}'|sort|uniq -c|sort -nr|less

11、统计每小时的请求数,top10的时间点(精确到小时)

    awk '{print $4}' /var/log/nginx/access.log |cut -c 14-15|sort|uniq -c|sort -nr|head -n 10

12、统计每分钟的请求数,top10的时间点(精确到分钟)

    awk '{print $4}' /var/log/nginx/access.log |cut -c 14-18|sort|uniq -c|sort -nr|head -n 10

13、统计每秒的请求数,top10的时间点(精确到秒)

    awk '{print $4}' /var/log/nginx/access.log |cut -c 14-21|sort|uniq -c|sort -nr|head -n 10

14、查找指定时间段的日志

    awk '$4 >="[25/Mar/2020:01:00:00" && $4 <="[25/Mar/2020:08:00:00"' /var/log/nginx/access.log

15、列出传输时间超过 0.6 秒的url，显示前10条

    cat /var/log/nginx/access.log |awk '(substr($NF,2,5) > 0.6){print $4,$7,substr($NF,2,5)}' | awk -F '"' '{print $1,$2,$3}' |sort -k3 -rn | head -10

16、列出/api/appid请求时间超过0.6秒的时间点

    cat /var/log/nginx/access.log |awk '(substr($NF,2,5) > 0.6 && $7~/\/api\/appid/){print $4,$7,substr($NF,2,5)}' | awk -F '"' '{print $1,$2,$3}' |sort -k3 -rn | head -10

17、获取前10条最耗时的请求时间、url、耗时

    cat /var/log/nginx/access.log |awk '{print $4,$7,substr($NF,2,5)}' | awk -F '"' '{print $1,$2,$3}' | sort -k3 -rn | head -10


## 参考文章

https://www.infoq.cn/article/esba98chggwexxmkqrhq

