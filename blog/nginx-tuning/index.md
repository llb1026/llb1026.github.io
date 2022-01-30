---
order: 2
layout: post
title: "혼자 해 본 nginx.conf 튜닝"
subtitle: "프론트엔드 최적화 포인트와 간단한 성능테스트 결과"
tag: Lesson Learned
type: lesson-learned
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../http-1.1-vs-2.0/"
prev-link: "../cookie-and-token/"
---

## 프론트엔드 최적화 포인트

### Static한 콘텐츠 캐싱

Nginx에서의 캐싱은 아래와 같이 동작한다.

1. Nginx가 static 리소스 요청을 받음
2. 요청받은 URL을 key로 해당 리소스가 캐시되어 있는지 확인
3. 첫 요청이므로 캐시 miss
4. WAS로 요청을 보냄
5. WAS로부터 응답을 받으면 캐싱할지 검사
6. 응답을 디스크에 저장해 캐싱
7. 다음번부터 같은 URL로 요청이 들어오면 캐시 hit, 해당 요청은 WAS로 전달되지 않고 Nginx가 대신 응답을 돌려줌

### Nginx as reverse proxy

Nginx가 reverse proxy로 동작하는 경우, static 리소스는 Nginx에서 캐싱해서 제공하고 dynamic 리소스는 proxy_pass를 통해 받은 요청을 upstream에 정의된 WAS로 넘긴다. 이렇게 구성할 경우 이점은 다음과 같다.

- **로드밸런싱**
    - Nginx가 받은 요청을 여러 WAS로 분산시켜 보낼 수 있다.
- **요청 선 처리**
    - WAS까지 요청이 가기 전에 필요한 선처리를 간단한 설정으로 구현할 수 있다.
    - 예를 들어, client validation을 위해 HTTP header의 user-agent나 referer를 검사하는 작업을 WAS에서 수행하려면 header 추출하고 비교하는 로직을 직접 구현해야 한다.
- **static 리소스의 빠른 응답**
    - WAS까지 가는 요청이 줄어든다.
    - Nginx나 Apache같은 전용 웹서버는 파일 읽기 연산에 최적화되어 있어서 WAS가 수행하는 것보다 빠르고 효율적으로 동작한다.
    - static 리소스와 dynamic 리소스를 분리해 처리함으로써 최적화가 쉬워진다.

### Nginx micro caching

micro caching은 WAS로 요청하는 dynamic 리소스에 대해 아주 짧은 시간동안 캐싱을 해서 WAS로 보내는 요청을 줄이는 방법이다. 사실 dynamic 리소스라고 하더라도 초단위로 페이지가 업데이트되는 경우는 드물어서, 1초라도 캐싱을 해 두면 많은 트래픽의 요청을 효과적으로 처리할 수 있게 된다.

기본적인 reverse proxy로 설정된 데에 micro caching을 추가한 설정은 다음과 같다.

```conf
# nginx.conf

proxy_cache_path /tmp/cache             // 캐시 파일이 로컬에 저장될 위치 설정
                 levels=1:2             // 캐시 파일을 어떻게 저장할지 결정 (첫번째 depth의 디렉토리 이름은 1글자, 두번째 depth의 디렉토리 이름은 2글자로 하겠다고 명시)
                 keys_zone=my_cache:10m // 캐시 이름 할당
                 inactive=600s          // 일정 시간동안 사용되지 않은 캐시 파일은 삭제하도록 설정
                 max_size=1g;           // 캐시 파일의 최대 크기 지정

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_cache my_cache;           // 사용할 캐시 지정
        proxy_cache_valid 200 1s;       // 1초동안 micro caching
        add_header X-Proxy-Cache $upstream_cache_status;  // 캐시 HIT, MISS, MYPASS와 같은 캐시 적중 상태정보 표시

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        ...
    }
}
```

단, micro caching을 도입하기 전에 고려해야 할 사항들이 있다.

- dynamic 리소스를 캐싱을 통해 제공하는 것보다 WAS에서 받는 것이 더 빠른 경우도 있다. 이러면 micro caching이 오히려 더 낮은 성능을 내는 요인이 될 수 있다.
- dynamic 리소스가 1초(혹은 개별적으로 정한 micro 단위시간)동안 캐싱되어서 내용이 변경되지 않아도 서비스에 영향이 없는지 확실한 검증이 필요하다.

### gzip

- `gzip_comp_level`: 1~9까지 설정 가능, 숫자가 클수록 압축률은 올라가지만 압축 속도는 느려짐
- `gzip_min_length`: 압축을 적용할 컨텐츠의 최소 사이즈 지정, 이보다 작은 파일은 압축하지 않음
- `gzip_buffers`: 버퍼의 숫자와 크기 지정
- `gzip proxyed`: proxy나 캐시 서버에서 요청할 경우 동작 여부 설정
    - `off`: proxy에서 요청할 경우 압축하지 않음
    - `expired`: 요청 header에 Expires가 있고 만료되었을 경우에만 압축
    - `no-cache`: 요청 header에 Cache-Control이 있고 no-cache일 경우에만 압축
    - `no-store`: 요청 header에 Cache-Control이 있고 no-store일 경우에만 압축
    - `any`: 항상 압축

### 최종 nginx.conf

```conf
user nginx;
worker_processes auto;
error_log /var/log/nginx-error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    use epoll;  # epoll은 Linux에서 socket을 관리하는 데 사용하는 방식 중 하나로, poll과 select가 해당 프로세스에 연결된 모든 connection file을 스캔하는 데에 비해 epoll은 수천개의 file descriptor를 처리할 수 있도록 보다 효율적인 알고리즘을 사용해 대량 요청이 발생하는 시스템에 적합하다
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # HDD의 IO 성능을 향상시키기 위해 보통 off로 두지만, 개발하면서 볼 용도로 켜놓음
    access_log  /var/log/nginx-access.log main;

    # 파일의 메타 데이터 정보를 캐시한다 (파일 컨텐츠가 아니라 메타 데이터!) 트래픽이 많을 경우 퍼포먼스 향상 효과를 크게 볼 수 있다
    open_file_cache             max=200000 inactive=20s;
    open_file_cache_valid       30s;
    open_file_cache_min_uses    2;
    open_file_cache_errors      on;

    # 파일을 압축 설정 (네트워크를 통해 전송되는 데이터 크기를 줄이기 위함)
    gzip                on;
    gzip_comp_level     9;
    gzip_min_length     10240;
    gzip_proxied        any;
    gzip_types          text/plain
                        text/css
                        text/js
                        text/xml
                        text/javascript
                        application/javascript
                        applicaion/x-javascript
                        application/json
                        applicatoin/xml
                        application/rss+xml
                        image/svg+xml;

    sendfile                    on;  # 응답을 보낼 때 user-space의 buffer 메모리를 쓰지 않고 바로 kernel file buffer를 사용해 더 빠르다
    reset_timedout_connection   on;  # timeout 이후 바로 메모리에 남아있는 데이터를 삭제해 메모리 공간 확보
    client_body_timeout         10;  # 요청 타임아웃 (디폴트 60)
    send_timeout                2;   # 클라이언트 측에서 응답이 없으면 메모리 free (디폴트 60)
    keepalive_timeout           30;  # 해당 시간 이후 서버가 커넥션을 닫음 (디폴트 75)
    keepalive_requests          100000;   # keep-alive동안 클라이언트가 보낼 수 있는 요청 개수

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # 로컬 캐시 설정
    proxy_cache_path    /var/cache/nginx  # 캐시 파일이 저장될 경로
                        levels=1:2  # 캐시 파일을 어떻게 저장할지 결정 (첫번째 depth의 디렉토리 이름은 1글자, 두번째 depth의 디렉토리 이름은 2글자로 하겠다고 명시)
                        keys_zone=nginx_cache:1m  # 캐시 이름
                        max_size=10g  # 캐시 파일의 최대 크기 지정
                        inactive=60s  # 일정 시간동안 사용되지 않은 캐시 파일은 삭제하도록 설정
                        use_temp_path=off;  # 불필요한 카피 데이터들이 임시 공간에 저장되는 것을 방지

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        root         /home1/irteam/deploy/internship/frontend/build;
        index        index.html index.htm;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        proxy_cache nginx_cache;
        proxy_cache_revalidate on;
        proxy_cache_lock on;
        proxy_cache_key $host$uri;
        proxy_cache_valid 200 1m;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;

        location / {
            root /home1/irteam/deploy/internship/frontend/build;
            try_files $uri /index.html;
        }

        location /api {
            proxy_pass http://10.105.188.47:8080;
            charset utf-8;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
        }

        location ~* \.(?:manifest|appcache|html?|xml)$ {
            expires -1;
        }

        location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
            expires 1M;
            access_log off;
            add_header Cache-Control "public";
        }

        location ~* \.(?:css|js)$ {
            expires 1y;
            access_log off;
            add_header Cache-Control "public";
        }

        error_page 400 401 403 404 /404.html;
            location = /40x.html {
        }

        error_page 500 501 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```

---

## 성능테스트

네이버 사내 프론트엔드 테스트 툴을 사용해 개발한 어플리케이션 성능테스트를 돌려보았다.

### 테스트 환경

- 기사 수: 12개
- 캐시: off
- 에이전트: Chrome_WIN10_I7
- 네트워크 환경: FTTH(100M) - 2ms, 91Mbps, 94Mbps

### 결과

<table class="uk-table-small uk-table style-2 uk-table-striped uk-text-center">
    <thead>
        <tr>
            <th style="text-align: center;"> \ </th>
            <th style="text-align: center;">최적화 이전</th>
            <th style="text-align: center;">최적화 이후</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><em>Time To First Byte</em></td>
            <td>17ms</td>
            <td>21ms</td>
        </tr>
        <tr>
            <td><em>Start Render</em></td>
            <td>600ms</td>
            <td>400ms</td>
        </tr>
        <tr>
            <td><em>Load Time</em></td>
            <td>580ms</td>
            <td>457ms</td>
        </tr>
        <tr>
            <td><em>Speed Index</em></td>
            <td>651ms</td>
            <td>448ms</td>
        </tr>
        <tr>
            <td><em>Visually Complete</em></td>
            <td>700ms</td>
            <td>500ms</td>
        </tr>
        <tr>
            <td><em>전체 로드한 크기</em></td>
            <td>849.8KB</td>
            <td>400.5KB</td>
        </tr>
        <tr>
            <td><em>요주의 리소스</em></td>
            <td>`0.chunk.js`: 509.8KB, <br />`main.chunk.js`: 130.5KB</td>
            <td>`0.chunk.js`: 73.1KB, <br />`main.chunk.js`: 3.7KB</td>
        </tr>
    </tbody>
</table>