[MackMd](https://hackmd.io/@pONMMCWcRMSASTBgLGLkyg/mru_datadog)

Data Dog文件
=======================

Created by mru huang, last modified on Feb 23, 2021

安裝說明
====

DataDog Agent
=============

data dog的監測服務，所以後續的服務都要先架agent起來讓其他服務與agent溝通後，再傳給datadog官網顯示數據

1.  dog
    
    1.  服務名稱，可以自訂修改名稱
        
2.  image
    
    1.  docker file的映像黨，此為datadog官方的docker hub檔案
        
3.  deploy
    
    1.  docker swarm的部屬訊息
        
    2.  會依照節點設定決定datadog agent會部屬在哪台機器上
        
4.  network
    
    1.  連結的網路設定
        
    2.  這邊只會去聯通datadog的network
        
    3.  其他服務如果想要被datadog監聽，再去設定該服務的network設定datadog連結
        
5.  volums
    
    1.  基本文件載入
        
    2.  特別注意`- ./datadog-conf/conf.d:/etc/datadog-agent/conf.d`
        
        1.  datadog-conf/conf.d是需要人工去修改相關監聽設定的檔案，非常重要
            
        2.  如果只安裝APM請註解該行
            
6.  environment
    
    1.  參數設定黨
        
        1.  `DD_API_KEY`為官網帳號提供的API KEY
            
            1.  請把KEY家到mmrm\_env的`DATADOG_API_KEY`設定
                
        2.  `DD_HOSTNAME`為容器的命名，會在APM看到的HOST名稱
            

docker swarm設置如下(已先註解conf.d的引用)
```
services:
## data dog ################
    dog:
      image: datadog/agent:latest
      deploy:
        mode: replicated
        replicas: 1
        placement:
          constraints:
            - node.labels.name == mmrm\_core
        restart\_policy:
          condition: on-failure
      networks:
        - datadog
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock:ro
        - /proc/:/host/proc/:ro
        - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
        #- ./datadog-conf/conf.d:/etc/datadog-agent/conf.d
      environment:
        - DD\_API\_KEY=${DATADOG\_API\_KEY}
        - DD\_HOSTNAME=datadog\_agent\_show\_name
        - DD\_APM\_ENABLED=true
        - DD\_TRACE\_AGENT\_PORT=8126
        - DD\_TRACE\_ANALYTICS\_ENABLED=true
        - DD\_APM\_NON\_LOCAL\_TRAFFIC=true
```
APM
---

監測API等流向、時間、佔用資源等服務

該服務為監控PHP程式流向為主，固主要會安裝在fpm服務上

1.  此服務安裝會需要安裝官方提供的php套件，已經寫docker file於`wishmobile/mmrm:php-fpm_datadog`，所以只要image引用即可
    
2.  需要特別設定的參數
    
    1.  `environment`
        
        1.  DD\_AGENT\_HOST=dog
            
            1.  dog為datadog agent服務的自訂名稱，需要該設定fpm才會知道要把數據傳給誰
                
            2.  如果安裝有另外的名稱需要加(例如mmrm\_外另名稱)
                
    2.  `networks`
        
        1.  datadog
            
            1.  要傳給datadog agent只能通過datadog network的network，所以要把datadog的網路加進去
                

    範例如下
    ```
     extension\_php-fpm:
          image: wishmobile/mmrm:php-fpm\_datadog0.55.0
          deploy:
            mode: replicated
            replicas: 1
            placement:
              constraints:
                - node.labels.name == mmrm\_extension
            restart\_policy:
              condition: on-failure
          volumes:
            - ${EXTENSION\_PHP\_CONFIG\_PATH}/php${PHP\_VERSION}.ini:/usr/local/etc/php/php.ini
            - ${EXTENSION\_SOURCE\_PATH}:/var/www:cached
            - ${EXTENSION\_PHP\_CONFIG\_PATH}/php-fpm.conf:/usr/local/etc/php-fpm.d/www.conf
            - ${EXTENSION\_PHP\_CONFIG\_PATH}/xlaravel.pool.conf:/usr/local/etc/php-fpm.d/xlaravel.pool.conf
            - ${APP\_API\_SOURCE\_PATH}:/var/www3:cached
            - ${POS\_API\_SOURCE\_PATH}:/var/www4:cached
            - ${THIRD\_PARTY\_VOUCHER\_SOURCE\_PATH}:/var/www7:cached
            - ${EXTENSION\_PHP\_CONFIG\_PATH}/docker.conf:/usr/local/etc/php-fpm.d/docker.conf
            - ${EXTENSION\_PHP\_CONFIG\_PATH}/log:/usr/local/var/log
          environment:
            - FAKETIME=${PHP\_FPM\_FAKETIME}
            - DD\_AGENT\_HOST=dog
          networks:
            - extension\_backend
            - core\_rabbitmq
            - core\_api
            - app\_api\_redis
            - app\_traffic\_limit\_redis
            - member\_redis
            - pos\_api\_backend
            - push\_api
            - member\_mysql
            - datadog
    ```
    上續兩項設定完成，將服務重啟後，應該可於datadog的網站上APM服務類別監測的到服務項目

其他監測服務(主要監控服務吃的資源與存活狀態、總網路用量等)
------------------------------

其餘的監測服務，都需要透過`datadog-conf/conf.d`之下的設定黨修改進行串接，主要步驟分為

1.  在con.d下會有各種服務的.d資料夾，裡面會有conf.yaml.example檔，如果想要啟動該服務的監測，cp為conf.yaml後修改設定參數即可
    
2.  在docker swarm該服務下加入network datadog的連結
    
3.  重啟服務，完成
    

### 各服務設定

1.  redis
    
```
instances:

    ## @param host - string - required
    ## Enter the host to connect to.
    #
  - host: extension\_redis

    ## @param port - integer - required
    ## Enter the port of the host to connect to.
    #
    port: 6379
    password: 請去docker yaml找這個password在哪然後改到這
```
2\. mysql

1.  要註冊一個帳號給datadog使用
    
```
`CREATE USER 'datadog'@'localhost' IDENTIFIED WITH mysql_native_password by '<UNIQUEPASSWORD>';`

`ALTER USER 'datadog'@'localhost' WITH MAX_USER_CONNECTIONS 5;`

`GRANT REPLICATION CLIENT ON *.* TO 'datadog'@'localhost' WITH MAX_USER_CONNECTIONS 5;`

`GRANT PROCESS ON *.* TO 'datadog'@'localhost';`
```
```
instances:

    ## @param server - string - required
    ## MySQL server to connect to.
    ## NOTE: Even if the server name is "localhost", the agent connects to MySQL using TCP/IP, unless you also
    ## provide a value for the sock key (below).
    #

  - server: mmrm\_core\_mysql

    ## @param user - string - required
    ## Datadog Username created to connect to MySQL.
    #
    user: datadog

    ## @param pass - string - required
    ## Password associated with the datadog user.
    #
    pass: datadog
```

3\. fpm

```
instances:

    ## @param status\_url - string - required
    ## Get metrics from your FPM pool with this URL
    ## The status URLs should follow the options from your FPM pool
    ## See http://php.net/manual/en/install.fpm.configuration.php
    ##   \* pm.status\_path
    ## You should configure your fastcgi passthru (nginx/apache) to
    ## catch these URLs and redirect them through the FPM pool target
    ## you want to monitor (FPM \`listen\` directive in the config, usually
    ## a UNIX socket or TCP socket.
    #
  - status\_url: http://mmrm\_extension\_php-fpm/phpfpm-status

    ## @param ping\_url - string - required
    ## Get a reliable service check of your FPM pool with \`ping\_url\` parameter
    ## The ping URLs should follow the options from your FPM pool
    ## See http://php.net/manual/en/install.fpm.configuration.php
    ##   \* ping.path
    ## You should configure your fastcgi passthru (nginx/apache) to
    ## catch these URLs and redirect them through the FPM pool target
    ## you want to monitor (FPM \`listen\` directive in the config, usually
    ## a UNIX socket or TCP socket.
    #
    ping\_url: http://mmrm\_extension\_php-fpm/php\_fpm\_ping

    ## @param use\_fastcgi - boolean - required - default: false
    ## Communicate directly with PHP-FPM using FastCGI
    #
    use\_fastcgi: true

    ## @param ping\_reply - string - required
    ## Set the expected reply to the ping.
    #
    ping\_reply: pong
```
4\. nginx

請注意在nginx的設定有`wish_nginx_status`的路徑(該laravel專案下nginx/sites-availble/app.conf)
```
instances:

    ## @param nginx\_status\_url - string - required
    ## For every instance, you need an \`nginx\_status\_url\` and can optionally
    ## supply a list of tags.  This plugin requires nginx to be compiled with
    ## the nginx stub status module option, and activated with the correct
    ## configuration stanza.  On debian/ubuntu, this is included in the
    ## \`nginx-extras\` package.  For more details, see:
    ##
    ##   http://docs.datadoghq.com/integrations/nginx/
    ##
    ## For NGINX Plus API Release 13+ users using the http\_api\_module,
    ## set nginx\_status\_url to the location of the \`/api\` endpoint. 
    ## E.g. \`nginx\_status\_url: http://localhost:8080/api\`
    #
  - nginx\_status\_url: http://mmrm\_extension\_nginx/wish\_nginx\_status/
```
