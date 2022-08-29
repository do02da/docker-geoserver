## Docker-compose + Apache HTTPD Web Server + Apache Tomcat * 2
---
### Apache Httpd 설정

- #### httpd.conf
```properties
#ServerName www.example.com:80 주석해제
ServerName localhost:80

Include conf/vhosts.conf
Include conf/mod_jk.conf
```

- #### vhosts.conf
```properties
# LoadModule jk_module modules/mod_jk.so : 모듈 mod_jk.so 불러오기
# <VirtualHost *:80> : 모든 IP주소에서 요청을 기다림
# ServerName localhost : server 이름 - 웹서버 주소
# JkUnmount /*.html tomcat_lb : *.html 부분은 tomcat_lb 에서 처리 안함 (apache에서 처리)
# JkMount /* tomcat_lb : 나머지 전부 tomcat_lb 에서 처리(workers.porperties 참고)
# </VirtualHost> 

LoadModule jk_module modules/mod_jk.so
<VirtualHost *:80>
  ServerName localhost
  # JkUnmount /*.html tomcat_lb
  JkMount /* tomcat_lb
</VirtualHost>
```

  - #### mod_jk.conf
```properties
# JkLogFile logs/mod_jk.log : log 저장
# JkLogStampFormat "[%a %b %d %H:%M:%S %Y]" : 날짜/시간을 스트링으로 변환
<ifModule jk_module>
  JkWorkersFile conf/workers.properties
  JkLogFile logs/mod_jk.log
  JkLogLevel info
  JkShmFile /var/log/httpd/jk-runtime-status
  JkWatchdogInterval 30
  JkLogStampFormat "[%a %b %d %H:%M:%S %Y]"
</ifModule>
```
  
- #### workers.properties
```properties
worker.list=tomcat_lb
worker.tomcat_lb.type=lb                            # lb == loadbalancer
worker.tomcat_lb.balance_workers=tomcat01,tomcat02  # loadbalancer instance 2개
worker.tomcat_lb.sticky_session=true
worker.tomcat_lb.session_cookie=JSESSIONID

worker.tomcat01.port=8009
worker.tomcat01.host=tomcat01
worker.tomcat01.type=ajp13
worker.tomcat01.lbfactor=1

worker.tomcat02.port=8009
worker.tomcat02.host=tomcat02
worker.tomcat02.type=ajp13
worker.tomcat02.lbfactor=1
```

---

## Tomcat 서버 설정
- ### server.xml
  * #### 주석 해제
    - `<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>`
    - `<Connector protocol="AJP/1.3" ~ />`
  * #### jvmRoute 추가 (tomcat01, tomcat02) : `<Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat01">`
  * #### SSL 미사용시 AJP Connector에 `secretRequired="false"` 추가

```xml
  <Connector protocol="AJP/1.3"
              address="::1"
              port="8009"
              redirectPort="8443"
              secretRequired="false" />

  <Engine name="Catalina" defaultHost="localhost" jvmRoute="tomcat01">
    <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
    <!-- 생략 -->
  </Engine>
```

- ### web.xml : `<distributable/>` 추가
```xml
<web-app>
  <!-- 생략 -->
  <distributable/>
</web-app>
```
--- 
## geoserver
* [GeoServer 공식 다운로드 홈페이지](https://geoserver.org/download/)에서 Web Archive (war) 파일 다운로드
* 압축 해제 후 geoserver.war 파일을 ROOT.war로 이름 변경
* Tomcat 10 사용시 `org.apache.catalina.core.StandardContext.listenerStart Error configuring application listener of class` 에러 발생 가능 ([참고](https://gis.stackexchange.com/questions/389555/geoserver-not-compatible-with-tomcat-10)) -> Tomcat 9 사용
    
---

## nginx
- ### AJP 대신 proxy pass 방식 사용 → nginx.conf 파일 참고
- ### docker-compose.yml에서 httpd 주석, nginx 주석 해제하여 사용 가능
 
---