# domain 설정하기 - TroubleShooting


## 문제 상황
- 가비아에서 domain을 구입하여, front-end 서버와 연결하였다.
- 이떄, Fixed Forwarding (고정 포워딩) 방식을 사용하였다.
- 또한 프론트 앤드는 서버에서 3000포트로 운영되고 있어 도메인을 바로 3000번 포트로 연결해주었다.
- 'http://serverIP:port/favicon.ico'로 접속시 아이콘이 잘 뜨지만, 'http://todayfin.site/faviconico' 연결 시 아이콘이 뜨지 않았다.
- 처음에는 favicon.ico의 경로 문제라고 생각하여 경로를 이리 저리 바꿨지만 해결되지 않았다.

## 문제 원인
- 현재 모든 서비스는 3000번 혹은 80번 포트 이외에 다른 포트를 이용하고 있어 serverIp:port 로 다이렉트로 도메인을 매핑하려고 하다 보니 고정 포워딩 방식을 사용했다.
- 고정 포워딩은 A 도메인 연결시 A 도메인을 B도메인으로 치환해주고 B도메인 요청 결과를 보여주면서 브라우저 창에는 A 도메인 주소만 뜨는 간단한 구조인 줄 알았지만, 고정 포워딩 방식을 사용할 경우 프로젝트 내의 디렉토리 구조가 변경될 수 있다는 것을 알게 되었다.
- 간단하게 말하자면 'http://todayfin.site/favicon.ico'로 요청을 보낸다면, 실제 요청은 'http://serverIP:3000/favicon.ico'로 전달되지만, 웹 서버 설정에 따라 경로 해석이 올바르게 이루어지지 않을 수도 있다는 것이 문제였다.


## 해결 방안
- 먼저 도메인을 특정 ip에 매핑해주는 'DNS관리' 서비스를 이용하기로 했다.
- 처음에 그냥 'http' 프로토콜로 특정 ip로 요청을 보내면 80번 포트로 자동으로 연결되는지 몰라서, 특정 포트(예: 3000번)를 열어 서비스를 하였다.
- 인터넷을 찾아보고 http 요청으로 특정 ip주소로 요청을 하면 자동으로 80번 포트로 연결되는 것을 알게 되었다.
- 그래서 가비아의 DNS 서비스를 이용하여 도메인을 서버 ip로 연결하고, 서버에서는 80포트를 열어놓고 nginx 같은 웹 서버를 리버스 프록시로 사용하여, 80포트로 오는 요청을 3000번 포트로 라우팅하여 문제를 해결하였다.
- 해결 순서
    1. 도메인과 IP 매핑
    2. 웹 서버 설정 변경


## 1. 도메인과 IP 매핑
- 내가 구입한 domain : todayfin.site
- 먼저 domain 이름을 특정 ip에 매핑하는 양식은 다음과 같다.
    ```
    type A :
    host.todayfin.site -> 특정 ip
    ```
- DNS 설정의 type에는 여러가지가 있다. (ex: A, MX, CNAME, TXT, SRV)
- 이중 domain을 ipv4주소로 매핑해주는 type A를 사용하였고 다음과 같이 설정하였다.

| type | Host | IP |
|----------|----------|----------|
| A | www | front-end Server IP |
| A | front | front-end Server IP |
| A | @ | front-end Server IP |
| A | back | back-end Server IP |

- @로 설정을 할 경우 'http://todayfin.site'로 오는 모든 요청을 해당 ip로 매핑한다.
- 앞에 host 이름을 붙여 여러개의 서버 ip를 붙일 수 있는 것을 알게 되어 프론트 서버와 백 서버도 매핑할 수 있게 되었다.
- 결론적으로 'www.' 'front.' + 'todayfin.site' 혹은 'todayfin.site' 로 연결 시 프론트앤드 서버의 80포트로 연결이 되며, 'back.todayfin.site'로 연결 시 백앤드 서버의 80포트로 연결이 된다. (HTTP 프로토콜 사용시)
- 그리고 당연히 ':특정Port번호' 를 사용하여 서버의 특정 포트에도 연결할 수 있다.


## 2. 웹 서버 설정 변경
- 이제 프론트 서버에서 80포트로 오는 요청을 내가 운영하는 프론트앤드 서버의 포트번호(3000)로 매핑하면된다.
- 여기서는 nginx라는 웹 서버의 리버스 프록시 기능을 사용하였다.

> 리버스 프록시는 특정 포트로 오는 요청을 다른 서버나 다른 포트로 리다이렉트하는 것을 의미한다.

- 나중에 문서화하여 여러 서버에서도 이용하면 좋기 때문에, Docker를 이용하여 nginx를 띄우고 80포트로 포트포워딩해서 운영하기로 했다.
- nginx.conf
    ```
    worker_processes 1;

    events {
        worker_connections 1024;
    }

    http {
        sendfile on;

        server {
            listen 80;
            server_name www.todayfin.site;

            location / {
                proxy_pass http://192.168.1.94:3000;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }
    ```
    - listen 80 부분은 80번 포트를 리스닝하고 있다는 의미다.
    - location의 proxy_pass 를 통해 어디로 라우팅할지 정하면 된다.
    - 참고로 도커를 사용하지 않으면 "http://localhost:3000"으로 라우팅하면 되지만, 도커를 사용하기 때문에 서버의 내부 ip주소를 사용하였다.
    - 왜냐하면 도커에서 로컬호스트를 사용하면 실행되는 도커 컨테이너 내부로 요청이 들어오기 때문이다.

- Dockerfile
    ```
    FROM nginx:alpine

    # Remove the default Nginx configuration file
    RUN rm /etc/nginx/conf.d/default.conf

    # Copy the configuration file from the current directory to the container
    COPY nginx.conf /etc/nginx/nginx.conf

    EXPOSE 80

    # Start Nginx
    CMD ["nginx", "-g", "daemon off;"]
    ```

- 이미지 빌드 및 실행
    ```
    $ docker build -t mynginx .
    $ docker run -d -p 80:80 --name mynginx mynginx
    ```



