# Traefik v2 </br>guide by examples

requirements

- have docker running somewhere
- have a domain `whateverblablabla.org`
- use cloudflare to manage DNS of the domain
- have 80/443 ports open

chapters

1. [traefik routing to docker containers](#1-traefik-routing-to-various-docker-containers)
2. [traefik routing to a local IP addresses](#2-traefik-routing-to-a-local-IP-addresses)
3. [middlewares](#3-middlewares)
4. [let's encrypt certificate HTTP challenge](#4-lets-encrypt-certificate-HTTP-challenge)
5. [let's encrypt certificate DNS challenge](#5-lets-encrypt-certificate-DNS-challenge-on-cloudflare)
6. [redirect HTTP traffic to HTTPS](#6-redirect-HTTP-traffic-to-HTTPS)

# #1 traefik routing to various docker containers

![traefik-dashboard-pic](https://i.imgur.com/5jKHJmm.png)

- **create a new docker network** `docker network create traefik_net`.</br>
  Traefik and the containers need to be on the same network.
  Compose creates one automatically, but that fact is hidden and there is potential for a fuck up later on.
  Better to just create own network and set it as default in every compose file.

  *extra info:* use `docker network inspect traefik_net` to see containers connected to that network

- **create traefik.yml**</br>
  This file contains so called static traefik configuration.</br>
  In this basic example there is log level set, dashboard is enabled.
  Entrypoint called `web` is defined at port 80. Docker provider is set and given docker socket</br>
  Since exposedbydefault is set to false, a label `"traefik.enable=true"` will be needed
  for containers that should be routed by traefik.</br>
  This file will be passed to a docker container using bind mount,
  this will be done when we get to docker-compose.yml for traefik.

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
    ```

  later on when traefik container is running, use command `docker logs traefik` 
  and check if there is a notice stating: `"Configuration loaded from file: /traefik.yml"`.
  You don't want to be the moron who makes changes to traefik.yml
  and it does nothing because the file is not actually being used.

- **create `.env`** file that will contain environmental variables.</br>
  Domain names, api keys, ip addresses, passwords,... 
  whatever is specific for one case and different for another, all of that ideally goes here.
  These variables will be available for docker-compose when running
  the `docker-compose up` command.</br>
  This allows compose files to be moved from system to system more freely and changes are done to the .env file,
  so there's a smaller possibility for a fuckup of forgetting to change domain name
  in some host rule in a big ass compose file or some such.

    `.env`
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```

  *extra info:*</br>
  command `docker-compose config` shows how compose will look
  with the variables filled in.
  Also entering container with `docker container exec -it traefik sh`
  and then `printenv` can be useful.

- **create traefik-docker-compose.yml file**.</br>
  It's a simple typical compose file.</br>
  Port 80 is mapped since we want traefik to be in charge of what comes on it - using it as an entrypoint.
  Port 8080 is for dashboard where traefik shows info. Mount of docker.sock is needed,
  so it can actually do its job interacting with docker.
  Mount of `traefik.yml` is what gives the static traefik configuration.
  The default network is set to the one created in the first step, as it will be set in all other compose files.

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **run traefik-docker-compose.yml**</br>
  `docker-compose -f traefik-docker-compose.yml up -d` will start the traefik container.

  traefik is running, you can check it at the ip:8080 where you get the dashboard.</br>
  Can also check out logs with `docker logs traefik`.

  *extra info:*</br>
  Typically you see guides having just a single compose file called `docker-compose.yml`
  with several services/containers in it. Then it's just `docker-compose up -d` to start it all.
  You don't even need to bother defining networks when it is all one compose.
  But this time I prefer small and separate steps when learning new shit.
  So that's why going with custom named docker-compose files as it allows easier separation.

  *extra info2:*</br>
  What you can also see in tutorials is no mention of traefik.yml
  and stuff is just passed from docker-compose using traefik's commands or labels.</br>
  like [this](https://docs.traefik.io/getting-started/quick-start/): `command: --api.insecure=true --providers.docker`</br>
  But that way compose files look bit more messy and you still can't do everything from there,
  you still sometimes need that damn traefik.yml.</br>
  So... for now, going with nicely structured readable traefik.yml

- **add labels to containers that traefik should route**.</br>
  Here are examples of whoami, nginx, apache, portainer.</br>

  > \- "traefik.enable=true"

  enables traefik

  > \- "traefik.http.routers.whoami.entrypoints=web"

  defines router named `whoami` that listens on entrypoint web(port 80)

  > \- "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

  defines a rule for this `whoami` router, specifically that when url
  equals `whoami.whateverblablabla.org` (the domain name comes from the `.env` file),
  that it means for router to do its job and route it to a service.
  
  Nothing else is needed, traefik knows the rest from the fact that these labels
  are coming from context of a docker container.

  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=web"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `apache-docker-compose.yml`
  ```
  version: "3.7"

  services:
    apache:
      image: httpd:latest
      container_name: apache
      hostname: apache
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.apache.entrypoints=web"
        - "traefik.http.routers.apache.rule=Host(`apache.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `portainer-docker-compose.yml`
  ```
  version: "3.7"

  services:
    portainer:
      image: portainer/portainer
      container_name: portainer
      hostname: portainer
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock:ro
        - portainer_data:/data
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.entrypoints=web"
        - "traefik.http.routers.portainer.rule=Host(`portainer.$MY_DOMAIN`)"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK

  volumes:
    portainer_data:

  ```

- **run the damn containers**</br>
  ignore some orphans talk, it's cuz these compose files are in the same directory
  and compose uses parent directory name for name of compose project

    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>
    `docker-compose -f apache-docker-compose.yml up -d`</br>
    `docker-compose -f portainer-docker-compose.yml up -d`

  *extra info:*</br>
  to stop all containers running: `docker stop $(docker ps -q)`

# #2 로컬 IP 주소로 트래픽 라우팅


URL이 도커 컨테이너가 아닌 다른 것을 목표로 해야 하는 경우.


![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)


- **파일 공급자 정의, 필수 라우팅 및 서비스 추가**
  
필요한 것은 일부 URL을 포착하여 일부 IP로 라우팅하는 라우터입니다.</br>
이전 예제에서는 포트 80에서 어떤 URL을 잡는 방법을 보여주었습니다,
하지만 아무도 규칙에 맞는 일이 발생했을 때 어떻게 해야 하는지 알려주지 않았습니다.
Traefik은 컨테이너의 컨텍스트에서 레이블을 사용하여 모든 작업을 수행했기 때문에 알고 있었습니다.
`traefik.yml`에서 도커를 공급자로 설정한 덕분입니다.</br
이 '특정 IP에서 트래픽 전송'을 위해서는 Traefik 서비스가 필요합니다,
그리고 트라픽 서비스를 정의하려면 새로운 제공자, 즉 파일 제공자가 필요합니다.
파일에 트래픽이 수행해야 할 작업을 알려줍니다.</br>
`traefik을 설정하는 것이 다소 일반적입니다.yml` 자체를 파일 공급자로 설정하는 것입니다.</br>
공급자 아래에는 새로운 `파일` 섹션과 `traefik이 있습니다.yml` 자체가 설정됩니다.</br>
그런 다음 동적 구성 항목이 추가됩니다.</br>
간단한 하위 도메인 호스트 이름 규칙이 있는 `route-to-local-ip`라는 라우터입니다.
이 경우 정확한 URL `test.whateverblablabla.org`이 해당 규칙에 맞습니다,
를 로드밸런서 서비스로 보내면 특정 IP와 특정 포트로만 라우팅합니다.

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
      file:
        filename: "traefik.yml"

    ## DYNAMIC CONFIGURATION
    http:
      routers:
        route-to-local-ip:
          rule: "Host(`test.whateverblablabla.org`)"
          service: route-to-local-ip-service
          priority: 1000
          entryPoints:
            - web

      services:
        route-to-local-ip-service:
          loadBalancer:
            servers:
              - url: "http://10.0.19.5:80"
    ```

라우터의 우선 순위는 매우 높은 값인 1000으로 설정되어 있습니다,
다른 모든 라우터를 능가합니다,
나중에 글로벌 http -> https 리디렉션을 수행하는 데 사용하는 것과 같은 것입니다.


*추가 정보 :*</br>
안타깝게도 `.env` 변수는 여기서 작동하지 않습니다,
그렇지 않으면 호스트 규칙의 도메인 이름과 해당 IP가 변수로 사용됩니다.
따라서 변경하는 것을 잊어버릴 수 있으니 주의하세요.


- **run traefik-docker-compose** 그리고 작동하는지 테스트합니다.

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    
# #3 middlewares

모든 컨테이너에 대한 인증 미들웨어의 예입니다.


![logic-pic](https://i.imgur.com/QkfPYel.png)


- **새 파일 만들기 - -사용자 자격증명사용자 자격증명-`users_credentials`** 사용자명이 포함된 파일입니다:비밀번호 쌍을 포함합니다,
[htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style</br>
아래 예시는 3개의 계정 모두에 비밀번호 `krakatoa`가 설정되어 있습니다.

    `users_credentials`
    ```
    me:$apr1$L0RIz/oA$Fr7c.2.6R1JXIhCiUI1JF0
    admin:$apr1$ELgBQZx3$BFx7a9RIxh1Z0kiJG0juE/
    bastard:$apr1$gvhkVK.x$5rxoW.wkw1inm9ZIfB0zs1
    ```

- **mount users_credentials in traefik-docker-compose.yml**

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./users_credentials:/users_credentials:ro"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **인증이 있어야 하는 컨테이너에 두 개의 레이블을 추가합니다</br>
- 첫 번째 레이블은 `auth-middleware`라는 새 미들웨어를 첨부합니다.
를 이미 존재하는 `whoami` 라우터에 추가합니다.
- 두 번째 레이블은 이 미들웨어 유형에 기본값을 부여합니다,
를 호출하여 사용자 인증에 사용할 파일이 어디에 있는지 알려줍니다.


여기에 `users_credentials` 파일을 마운트할 필요가 없으며, 해당 파일이 필요한 것은 traefik입니다.
그리고 이 레이블은 TRAEFIK에 정보를 전달하는 방법입니다.
컨테이너의 컨텍스트에서.

  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=web"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
        - "traefik.http.routers.whoami.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
        - "traefik.http.routers.nginx.middlewares=auth-middleware"
        - "traefik.http.middlewares.auth-middleware.basicauth.usersfile=/users_credentials"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

- **이 망할 컨테이너를 실행하면 이제 로그인과 비밀번호가 필요합니다.

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>

# #4 let's encrypt certificate, HTTP 도전 과제

![letsencrypt-http-challenge-pic](https://i.imgur.com/yTshxC9.png)


프로세스에 대한 이해가 단순해졌습니다.


`LE` - Let's Encrypt. 무료 인증서를 제공하는 서비스</br>
`인증서` - 서버의 파일에 저장된 암호화 키입니다,
암호화 통신을 허용하고 신원을 확인합니다< /br> .
`ACME` - 인증서를 협상하기 위한 프로토콜(정확하게 합의된 통신 방식)
에서. traefik의 일부입니다.</br>
`DNS` - 인터넷의 서버, 도메인 이름을 IP 주소로 변환</br>

Traefik은 ACME를 사용하여 `whateverblablabla.org`와 같은 특정 도메인에 대한 인증서를 LE에 요청합니다.
LE는 트라픽이 서버의 특정 위치에 무작위로 생성한 텍스트로 응답합니다.
그런 다음 LE는 DNS 인터넷 서버에 `whateverblablabla.org`가 어떤 IP 주소를 가리키고 있는지 요청합니다.
LE는 포트 80/443을 통해 임의의 텍스트가 포함된 파일에서 해당 IP 주소를 찾습니다.

인증서가 있다면 이는 인증서를 요청한 사람이 두 인증서를 모두 제어하고 있음을 증명합니다.
서버와 도메인에 대한 제어권을 보여줬기 때문입니다.
인증서가 제공되며 3개월 동안 유효하며, traefik은 자동으로 갱신을 시도합니다.
남은 기간이 30일 미만인 경우


이제 실제로 완료하는 방법을 알아보세요.


- **600개의 권한이 있는 빈 acme.json 파일을 만듭니다**


이 파일에는 인증서와 인증서에 대한 모든 정보가 저장됩니다.

  `touch acme.json && chmod 600 acme.json`

- **443 진입점 및 인증서 확인자를 TRAEFIK에 추가합니다.yml**</br>


엔트리포인트 섹션에 웹시큐어(포트 443)라는 새 엔트리포인트가 추가되었습니다.
certificatesResolvers는 traefik에 다음을 알려주는 구성 섹션입니다.
ACME 해결사를 사용하여 인증서를 받는 방법.

    ```
    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

- 리졸버의 이름은 `lets-encr`이고 acme을 사용합니다.
- 스테이징 caServer를 주석 처리하면 LE가 스테이징 인증서를 발급합니다,
유효하지 않은 인증서이며 녹색 잠금을 제공하지 않지만 제한은 없습니다,
따라서 테스트하기에 좋습니다. 작동하면 암호화하자에서 발행했다고 표시됩니다.
- 저장소는 주어진 인증서를 저장할 위치를 알려줍니다 - `acme.json`
- 이메일은 LE가 만료되는 인증서에 대한 알림을 보내는 곳입니다.
- httpChallenge에 진입점이 주어지므로 acme는 포트 80을 통해 http 챌린지를 수행합니다.


이것이 최고를 위해 필요한 전부입니다.

    `traefik.yml`
    ```
    ## STATIC CONFIGURATION
    log:
      level: INFO

    api:
      insecure: true
      dashboard: true

    entryPoints:
      web:
        address: ":80"
      websecure:
        address: ":443"

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false

    certificatesResolvers:
      lets-encr:
        acme:
          #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
          storage: acme.json
          email: whatever@gmail.com
          httpChallenge:
            entryPoint: web
    ```

- **expose/map 포트 443 및 traefik-docker-compose.yml에 acme.json 마운트**


acme.json이 **not** :ro - 읽기 전용이라는 점에 유의하세요.

    `traefik-docker-compose.yml`
    ```
    version: "3.7"

    services:
      traefik:
        image: "traefik:v2.1"
        container_name: "traefik"
        hostname: "traefik"
        env_file:
          - .env
        ports:
          - "80:80"
          - "443:443"
          - "8080:8080"
        volumes:
          - "/var/run/docker.sock:/var/run/docker.sock:ro"
          - "./traefik.yml:/traefik.yml:ro"
          - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **용기에 필수 레이블 추가**</br>
첫 번째 장의 일반 HTTP와 비교해보십시오,
라우터의 엔트리포인트를 `web에서 `로 변경하는 것뿐입니다. class="ͼ38">`에서 `websecure`로 변경하는 것입니다.
라는 이름의 인증서 확인자를 기존 라우터에 할당합니다. `lets-encr`

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```
- **그 망할 컨테이너를 실행하세요**</br>
잠시만 기다려주세요</br>
컨테이너는 이제 https를 통해서만 작동하며 greenlock< /br> 을 갖습니다.


*추가 정보 :*</br>
acme.json의 내용 확인</br>
새로 시작하려면 acme.json을 삭제하세요.
# #5 클라우드플레어에서 인증서 DNS 챌린지를 암호화하자

![letsencrypt-dns-challenge-pic](https://i.imgur.com/dkgxFTR.png)

프로세스에 대한 이해가 단순해졌습니다.


`LE` - Let's Encrypt. 무료 인증서를 제공하는 서비스</br>
`인증서` - 서버의 파일에 저장된 암호화 키입니다,
암호화 통신을 허용하고 신원을 확인합니다< /br> .
`` - 인증서를 협상하기 위한 프로토콜(정확하게 합의된 통신 방식)
에서. traefik의 일부입니다.</br>
`DNS` - 인터넷의 서버, 도메인 이름을 IP 주소로 변환</br&t;;


Traefik은 ACME를 사용하여 `whateverblablabla.org`와 같은 특정 도메인에 대한 인증서를 LE에 요청합니다.
LE는 무작위로 생성된 텍스트로 응답하며, traefik은 이 텍스트를 새 DNS TXT 레코드로 저장합니다.
그런 다음 LE는 `whateverblablabla.org` DNS 레코드를 확인하여 텍스트가 존재하는지 확인합니다.
  
인증서가 있으면 인증서를 요청한 사람이 도메인을 제어하고 있다는 것을 증명합니다.
인증서가 발급되며 3개월 동안 유효합니다. Traefik에서 자동으로 갱신을 시도합니다.
남은 기간이 30일 미만인 경우


와일드카드 인증서를 사용할 수 있다는 점이 httpChallenge보다 유리합니다.
모든 하위 도메인의 유효성을 검사하는 인증서입니다. `*.whateverblablabla.org`</br> .
또한 포트가 열려 있을 필요도 없습니다.


하지만 Traefik은 DNS 레코드를 자동으로 변경할 수 있어야 합니다,
따라서 사이트 DNS를 관리하는 사람이 이를 지원해야 합니다.
그렇기 때문에 Cloudflare를 선택해야 합니다.


이제 실제로 완료하는 방법을 알아보세요.


- **계획된 모든 하위 도메인에 대해 유형 A DNS 레코드 추가**


whoami, nginx, \*

- **600개의 권한이 있는 빈 acme.json 파일을 만듭니다**


`touch acme.json && chmod 600 acme.json`


- **443 진입점 및 인증서 확인자를 traefik에 추가합니다.yml**</br>


엔트리포인트 섹션에 웹시큐어(포트 443)라는 새 엔트리포인트가 추가되었습니다.
certificatesResolvers는 traefik에 다음을 알려주는 구성 섹션입니다.
ACME 해결기를 사용하여 인증서를 얻는 방법.< /br>

    ```
    certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
    ```

- 리졸버의 이름은 `lets-encr`이고 acme을 사용합니다.
- 스테이징 caServer를 주석 처리하면 LE가 스테이징 인증서를 발급합니다,
유효하지 않은 인증서이며 녹색 잠금을 제공하지 않지만 제한은 없습니다,
작동 중이면 암호화하자에서 발행했다고 표시됩니다.
- 저장소는 주어진 인증서를 저장할 위치를 알려줍니다 - `acme.json`
- 이메일은 LE가 인증서 만료에 대한 알림을 보내는 곳입니다.
- dnsChallenge는 [제공자](https://docs.traefik.io/https/acme/#providers),
이 경우 클라우드플레어입니다. 각 공급자는 서로 다른 이름의 환경 변수가 필요합니다.
를 추가해야 하지만 이는 나중에 설명하며, 여기서는 공급자 이름만 있으면 됩니다.
- 확인자는 챌린지 중에 사용할 잘 알려진 DNS 서버의 IP입니다.

  `traefik.yml`
  ```
  ## STATIC CONFIGURATION
  log:
    level: INFO

  api:
    insecure: true
    dashboard: true

  entryPoints:
    web:
      address: ":80"
    websecure:
      address: ":443"

  providers:
    docker:
      endpoint: "unix:///var/run/docker.sock"
      exposedByDefault: false

  certificatesResolvers:
    lets-encr:
      acme:
        #caServer: https://acme-staging-v02.api.letsencrypt.org/directory
        email: whatever@gmail.com
        storage: acme.json
        dnsChallenge:
          provider: cloudflare
          resolvers:
            - "1.1.1.1:53"
            - "8.8.8.8:53"
  ```

- **에 `를 추가합니다.env` 파일에 필수 변수 추가**</br>
[지원되는 제공업체 목록](https://docs)를 기반으로 어떤 변수를 추가할지 알 수 있습니다.traefik.io/https/acme/#providers).</br>
클라우드플레어 변수는 다음과 같습니다.
- `CF_API_EMAIL` - 클라우드플레어 로그인
- `CF_API_KEY` - 글로벌 API 키

  `.env`
  ```
  MY_DOMAIN=whateverblablabla.org
  DEFAULT_NETWORK=traefik_net
  CF_API_EMAIL=whateverbastard@gmail.com
  CF_API_KEY=8d08c87dadb0f8f0e63efe84fb115b62e1abc
  ```

- **expose/map 포트 443 및 마운트 acme.json**을 traefik-docker-compose.yml에 추가합니다.


acme.json이 **not** :ro - 읽기 전용이라는 점에 유의하세요.

  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

- **컨테이너에 필수 레이블 추가**</br>
첫 번째 장의 일반 HTTP와 비교하여
- 라우터의 엔트리포인트가 `web로 전환됩니다. class="ͼe">`에서 `websecure`로 전환됩니다.
<라우터에 할당된 `lets-encr`이라는 이름의 인증서 확인자
- 인증서를 받을 기본 도메인을 정의하는 레이블입니다,
여기서는 누가미.whateverblablabla.org이며, `.env` 파일에서 가져온 도메인 이름입니다.
  `whoami-docker-compose.yml`
  ```
  version: "3.7"

  services:
    whoami:
      image: "containous/whoami"
      container_name: "whoami"
      hostname: "whoami"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.entrypoints=websecure"
        - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
        - "traefik.http.routers.whoami.tls.certresolver=lets-encr"
        - "traefik.http.routers.whoami.tls.domains[0].main=whoami.$MY_DOMAIN"


  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

  `nginx-docker-compose.yml`
  ```
  version: "3.7"

  services:
    nginx:
      image: nginx:latest
      container_name: nginx
      hostname: nginx
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.entrypoints=websecure"
        - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
        - "traefik.http.routers.nginx.tls.certresolver=lets-encr"
        - "traefik.http.routers.nginx.tls.domains[0].main=nginx.$MY_DOMAIN"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```
- **그 망할 컨테이너를 실행하세요**</br>
  `docker-compose -f traefik-docker-compose.yml up -d`</br>
  `docker-compose -f whoami-docker-compose.yml up -d`</br>
  `docker-compose -f nginx-docker-compose.yml up -d`</br>

- **제길, DNS 챌린지의 핵심은 와일드카드를 얻는 것입니다!**</br>
충분히 공정</br>
따라서 와일드카드의 경우 이 레이블은 traefik 컴포지션에 들어갑니다.
- 이전과 동일한 `lets-encr` 인증자 해결사가 사용되며, traefik.yml에 정의된 것과 동일합니다.
- 하위 도메인에 대한 와일드카드(*.whateverblablabla.org)가 인증서를 받을 기본 도메인으로 설정됩니다.
- 네이키드 도메인(그냥 무엇이든블라블라.org)은 sans(제목 대체 이름)로 설정됩니다.
다시 말하지만, `*.whateverblablabla.org` 및 `whateverblablabla.org`가 필요합니다.
DNS 제어판에서 traefik의 IP를 가리키는 A 레코드로 설정합니다.

  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
        - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
        - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```


이제 컨테이너를 하위 도메인으로 액세스하려는 경우,
URL에 대한 규칙이 있는 일반 라우터만 있으면 됩니다,
443 포트 진입점에 있어야 하며 동일한 `lets-encr` 인증서 확인자를 사용해야 합니다.

    `whoami-docker-compose.yml`
    ```
    version: "3.7"

    services:
      whoami:
        image: "containous/whoami"
        container_name: "whoami"
        hostname: "whoami"
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.whoami.entrypoints=websecure"
          - "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"
          - "traefik.http.routers.whoami.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

    `nginx-docker-compose.yml`
    ```
    version: "3.7"

    services:
      nginx:
        image: nginx:latest
        container_name: nginx
        hostname: nginx
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.nginx.entrypoints=websecure"
          - "traefik.http.routers.nginx.rule=Host(`nginx.$MY_DOMAIN`)"
          - "traefik.http.routers.nginx.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

다음은 아파치이지만 이번에는 네이키드 도메인 `whateverblablabla.org`</br
    
    `apache-docker-compose.yml`
    ```
    version: "3.7"

    services:
      apache:
        image: httpd:latest
        container_name: apache
        hostname: apache
        labels:
          - "traefik.enable=true"
          - "traefik.http.routers.apache.entrypoints=websecure"
          - "traefik.http.routers.apache.rule=Host(`$MY_DOMAIN`)"
          - "traefik.http.routers.apache.tls.certresolver=lets-encr"

    networks:
      default:
        external:
          name: $DEFAULT_NETWORK
    ```

# #6 http 트래픽을 https로 리디렉션


![padlocks-pic](https://i.imgur.com/twTDSym.png)


http(80)을 https(443)으로 리디렉션하는 것이 좋습니다.< /br>
Traefik에는 이를 위한 특별한 유형의 미들웨어인 redirectscheme이 있습니다.


이 리디렉션을 선언할 수 있는 곳은 여러 곳이 있습니다,
`traefik.yml`에서 동적 섹션에서 `traefik.yml` 자체가 파일 공급자로 설정됩니다.</br>
또는 실행 중인 컨테이너에서 레이블을 사용하거나, 이 예제에서는 traefik 컴포즈에서 레이블을 사용합니다.


- **트레픽 작성에서 레이블을 사용하여 새 라우터 및 리디렉션 체계를 추가**

  >\- "traefik.enable=true"

이 트라픽 컨테이너에서 트라픽을 활성화합니다,
여기에는 서비스에 대한 일반적인 라우팅이 필요하지 않습니다,
이 없으면 다른 레이블이 작동하지 않습니다.

  >\- "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"

라는 새 미들웨어를 생성합니다. `redirect-to-https`, "redirectscheme"을 입력합니다.
를 지정하고 `https`.</br>.

  >\- "traefik.http.routers.redirect-https.rule=hostregexp(`{host:.+}`)"

는 다음과 같은 정규식 규칙을 사용하여 `redirect-https`라는 새 라우터를 생성합니다.
모든 수신 요청을 포착합니다.

  >\- "traefik.http.routers.redirect-https.entrypoints=web"

이 라우터가 수신 대기하는 엔트리포인트를 선언합니다 - 웹(포트 80) </br>

  >\- "traefik.http.routers.redirect-https.middlewares=redirect-to-https"

는 새로 생성된 리디렉션스킴 미들웨어를 이 새로 생성된 라우터에 할당합니다.


요약하자면, 포트 80에서 요청이 들어오면 해당 엔트리포인트에서 수신 대기 중인 라우터가 이를 확인합니다.
규칙에 맞고 모든 것이 규칙에 맞으면 다음 단계로 넘어갑니다.
궁극적으로는 서비스에 도달해야 하지만 선언된 미들웨어가 있으면 해당 미들웨어가 먼저 도달합니다,
미들웨어가 있고 리디렉션 방식이기 때문에 어떤 서비스에도 도달하지 않습니다,
를 입력하면 포트 443으로 이동하라는 메시지가 표시되는 https 체계를 사용하여 리디렉션됩니다.


다음은 이전 장의 DNS 챌린지 레이블이 포함된 전체 트레픽 구성입니다:
    
  `traefik-docker-compose.yml`
  ```
  version: "3.7"

  services:
    traefik:
      image: "traefik:v2.1"
      container_name: "traefik"
      hostname: "traefik"
      env_file:
        - .env
      ports:
        - "80:80"
        - "443:443"
        - "8080:8080"
      volumes:
        - "/var/run/docker.sock:/var/run/docker.sock:ro"
        - "./traefik.yml:/traefik.yml:ro"
        - "./acme.json:/acme.json"
      labels:
        - "traefik.enable=true"
        ## DNS CHALLENGE
        - "traefik.http.routers.traefik.tls.certresolver=lets-encr"
        - "traefik.http.routers.traefik.tls.domains[0].main=*.$MY_DOMAIN"
        - "traefik.http.routers.traefik.tls.domains[0].sans=$MY_DOMAIN"
        ## HTTP REDIRECT
        - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
        - "traefik.http.routers.redirect-https.rule=hostregexp(`{host:.+}`)"
        - "traefik.http.routers.redirect-https.entrypoints=web"
        - "traefik.http.routers.redirect-https.middlewares=redirect-to-https"

  networks:
    default:
      external:
        name: $DEFAULT_NETWORK
  ```

- **그 망할 컨테이너를 실행하고 이제 `http://whoami.whateverblablabla.org`가 즉시 `https://whoami.whateverblablabla.org`로 변경됩니다.


# #결제할 내용
- [파일 공급자가 도커 컨테이너를 관리하는 데 사용되는 경우]https://github.com/pamendoz/personalDockerCompose)
- [traefik v2 포럼](https://community.containo.us/c/traefik/traefik-v2)
- ['traefik 공식 사이트 블로그](https://containo.us/blog/)
