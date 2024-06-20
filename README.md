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

# #2 traefik routing to a local IP addresses

  When url should aim at something other than a docker container.

  ![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)

- **define a file provider, add required routing and service**
  
  What is needed is a router that catches some url and route it to some IP.</br>
  Previous examples shown how to catch whatever url, on port 80,
  but no one told it what to do when something fits the rule.
  Traefik just knew since it was all done using labels in the context of a container and
  thanks to docker being set as a provider in `traefik.yml`.</br>
  For this "sending traffic at some IP" a traefik service is needed,
  and to define traefik service a new provider is required, a file provider - just a fucking stupid
  file that tells traefik what to do.</br>
  Somewhat common is to set `traefik.yml` itself as a file provider so thats what will be done.</br>
  Under providers theres a new `file` section and `traefik.yml` itself is set.</br>
  Then dynamic configuration stuff is added.</br>
  A router named `route-to-local-ip` with a simple subdomain hostname rule.
  What fits that rule, in this case exact url `test.whateverblablabla.org`,
  is send to the loadbalancer service which just routes it a specific IP and specific port.

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

  Priority of the router is set to 1000, a very high value,
  beating any possible other routers,
  like one we use later for doing global http -> https redirect.

  *extra info:*</br>
  Unfortunately the `.env` variables are not working here,
  otherwise domain name in host rule and that IP would come from a variables.
  So heads up that you will definitely forget to change these. 

- **run traefik-docker-compose** and test if it works

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    
# #3 middlewares

Example of an authentication middleware for any container.

![logic-pic](https://i.imgur.com/QkfPYel.png)

- **create a new file - `users_credentials`** containing username:passwords pairs,
 [htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style</br>
 Bellow example has password `krakatoa` set to all 3 accounts

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

- **add two labels to any container** that should have authentication</br>
  - The first label attaches new middleware called `auth-middleware`
    to an already existing `whoami` router.
  - The second label gives this middleware type basicauth,
    and tells it where is the file it should use to authenticate users.

    No need to mount the `users_credentials` here, it's traefik that needs that file
    and these labels are a way to pass info to traefik, what it should do
    in context of containers.

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

- **run the damn containers** and now there is login and password needed

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>

# #4 let's encrypt certificate, HTTP challenge

![letsencrypt-http-challenge-pic](https://i.imgur.com/yTshxC9.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a cryptographic key stored in a file on the server,
   allows encrypted communication and confirms the identity</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiate certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip address</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts at a specific place on the server.
  LE then asks DNS internet servers for `whateverblablabla.org` and that points to some IP address.
  LE looks at that IP address through ports 80/443 for the file containing that random text.

  If it's there then this proves that whoever asked for the certificate controls both
  the server and the domain, since it showed control over DNS records.
  Certificate is given and is valid for 3 months, traefik will automatically try to renew
  when less than 30 days is remaining.

  Now how to actually get it done.


- **create an empty acme.json file with 600 permissions**

  This file will store the certificates and all the info about them.

  `touch acme.json && chmod 600 acme.json`

- **add 443 entrypoint and certificate resolver to traefik.yml**</br>

  In entrypoint section new entrypoint is added called websecure, port 443
  
  certificatesResolvers is a configuration section that tells traefik
  how to use acme resolver to get certificates.

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

  - the name of the resolver is `lets-encr` and uses acme
  - commented out staging caServer makes LE issue a staging certificate,
    it is an invalid certificate and wont give green lock but has no limitations,
    so it's good for testing. If it's working it will say issued by let's encrypt.
  - Storage tells where to store given certificates - `acme.json`
  - The email is where LE sends notification about certificates expiring
  - httpChallenge is given an entrypoint, so acme does http challenge over port 80

  That is all that is needed for acme

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

- **expose/map port 443 and mount acme.json in traefik-docker-compose.yml** 

  Notice that acme.json is **not** :ro - read only

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

- **add required labels to containers**</br>
compared to just plain http from first chapter,
it is just changing router's entryPoint from `web` to `websecure`
and assigning certificate resolver named `lets-encr` to the existing router

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
- **run the damn containers**</br>
  give it a minute</br>
  containers will now work only over https and have the greenlock</br>

  *extra info:*</br>
  check content of acme.json</br>
  delete acme.json if you want fresh start
 
# #5 let's encrypt certificate DNS challenge on cloudflare

![letsencrypt-dns-challenge-pic](https://i.imgur.com/dkgxFTR.png)

  My understanding of the process, simplified.

  `LE` - Let's Encrypt. A service that gives out free certificates</br>
  `Certificate` - a cryptographic key stored in a file on the server,
   allows encrypted communication and confirms the identity</br>
  `ACME` - a protocol(precisely agreed way of communication) to negotiate certificates
  from LE. It is part of traefik.</br>
  `DNS` - servers on the internet, translate domain names in to ip address</br>

  Traefik uses ACME to ask LE for a certificate for a specific domain, like `whateverblablabla.org`.
  LE answers with some random generated text that traefik puts as a new DNS TXT record.
  LE then checks `whateverblablabla.org` DNS records to see if the text is there.
  
  If it's there then this proves that whoever asked for the certificate controls the domain.
  Certificate is given and is valid for 3 months. Traefik will automatically try to renew
  when less than 30 days is remaining.

  Benefit over httpChallenge is ability to have wild card certificates.
  These are certificates that validate all subdomains `*.whateverblablabla.org`</br>
  Also no ports are needed to be open.

  But traefik needs to be able to make these automated changes to DNS records,
  so there needs to be support for this from whoever manages sites DNS.
  Thats why going with cloudflare.

  Now how to actually get it done.

- **add type A DNS records for all planned subdomains**

  [whoami, nginx, \*] are used example subdomains, each one should have A-record pointing at traefik IP

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
