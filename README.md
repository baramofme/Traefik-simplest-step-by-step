# Traefik v2 </br>예시별 가이드

요구 사항


- 어딘가에서 도커가 실행 중입니다.
- 도메인이 `whateverblablabla.org`인 경우
- 클라우드플레어를 사용하여 도메인의 DNS를 관리합니다.
- 80/443 포트가 열려 있습니다.


챕터

1. [traefik 라우팅에서 다양한 도커 컨테이너로](#1-다양한-도커-컨테이너로-traefik을-라우팅)
2. [traefik 라우팅을 로컬 IP 주소로](#2-로컬-IP-주소로-traefik을-라우팅)
3. [미들웨어](#3-미들웨어)
4. [렛츠 암호화 인증서 HTTP 챌린지 하기](#4-렛츠-암호화-인증서-HTTP-챌린지-하기)
5. [클라우드플레어에서 렛츠 암호화 인증서 DNS 챌린지 하기](#5-클라우드플레어에서-렛츠-암호화-인증서-DNS-챌린지-하기)
6. [redirect HTTP traefik을 HTTPS로](#6-http-traefik을-https로-리디렉션)

# #1 다양한 도커 컨테이너로 traefik을 라우팅

![traefik-dashboard-pic](https://i.imgur.com/5jKHJmm.png)

- **새 도커 네트워크 만들기** `도커 네트워크 만들기 traefik_net`.</br>
  Traefik과 컨테이너는 동일한 네트워크에 있어야 합니다.
  작성하면 자동으로 생성되지만 그 사실이 숨겨져 있어 나중에 문제가 발생할 가능성이 있습니다.
  자체 네트워크를 생성하고 모든 작성 파일에서 기본값으로 설정하는 것이 좋습니다.


  *추가 정보:* 해당 네트워크에 연결된 컨테이너를 보려면 `docker network inspect traefik_net`을 사용한다.

- **create traefik.yml**</br>
  이 파일에는 소위 정적 traefik 구성이 포함되어 있습니다.</br>
  이 기본 예제에서는 로그 수준이 설정되어 있고 대시보드가 활성화되어 있습니다.
  `web`이라는 진입점이 포트 80에 정의됩니다. 도커 공급자가 설정되고 도커 소켓</br>이 지정됩니다.
  exposedbydefault가 false로 설정되어 있으므로 traefik 을 통해 라우팅해야 하는 컨테이너의 경우 `"traefik.enable=true"` 레이블이 필요하게 됩니다.</br>
  이 파일은 바인드 마운트를 사용하여 도커 컨테이너로 전달됩니다,
  이 작업은 traefik의 docker-compose.yml로 이동하면 완료됩니다.

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

  나중에 traefik 컨테이너가 실행 중일 때 `docker logs traefik` 명령을 사용합니다.
  그리고 클릭하고 다음과 같은 알림이 있는지 확인합니다: `"파일에서 구성을 로드했습니다: /traefik.yml"`.
  traefik.yml을 변경하는 바보가 되고 싶지 않으세요?
  파일이 실제로 사용되지 않기 때문에 아무것도 하지 않습니다.

- 환경 변수를 포함하는 **`.env` 파일을 만듭니다.**</br>
  도메인 이름, API 키, IP 주소, 비밀번호,...
  어떤 경우에는 특정하고 다른 경우에는 다른 것이 무엇이든, 모든 것이 여기에 이상적으로 들어갑니다.
  다음 변수는 docker-compose를 실행할 때 사용할 수 있습니다.
  `docker-compose up` 명령입니다.</br>
  이렇게 하면 작성 파일을 시스템 간에 더 자유롭게 이동하고 .env 파일을 변경할 수 있습니다,
  따라서 호스트 규칙에서 도메인 이름을 변경하는 것을 잊어버리는 등의 실수가 발생할 가능성이 적습니다.

    `.env`
    ```
    MY_DOMAIN=whateverblablabla.org
    DEFAULT_NETWORK=traefik_net
    ```


  *추가 정보 :*</br>
  명령 `docker-compose config`은 컴포짓이 어떻게 표시되는지 보여준다.
  변수를 채워 넣습니다.
  또한 `docker container exec -it traefik sh`로 컨테이너를 입력합니다.
  그리고 나서 `printenv`를 사용하면 유용할 수 있습니다.

- **create traefik-docker-compose.yml 파일**.</br>
  간단한 일반적인 작성 파일입니다.</br>
  포트 80을 매핑한 이유는 이 포트를 엔트리 포인트로 사용하여 traefik을 처리하도록 하기 위해서입니다.
  포트 8080은 traefik이 정보를 표시하는 대시보드용입니다. docker.sock의 마운트가 필요합니다,
  이것은 그것의 job을 사용하여 실제로 도커와 상호 작용할 수 있도록 합니다.
`traefik.yml`의 마운트는 정적 traefik 구성을 제공합니다.
  기본 네트워크는 다른 모든 작성 파일에서 설정되므로 첫 번째 단계에서 만든 네트워크로 설정됩니다.

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

- **traefik-docker-compose.yml 실행**</br>
  `docker-compose -f traefik-docker-compose.yml up -d`하면 traefik 컨테이너가 시작됩니다.

  traefik이 실행 중이면 대시보드가 표시되는 IP:8080에서 확인할 수 있습니다.< /br>
`docker logs traefik`으로 로그를 확인할 수도 있습니다.


  *추가 정보 :*</br>
  일반적으로 가이드에는 `docker-compose.yml`이라는 하나의 작성(compose) 파일에 여러 개의 서비스/컨테이너와 함께 생성합니다. 그런 다음 `docker-compose up -d`로 모든 것을 시작하기만 하면 됩니다.
  하나의 컴포지션으로 네트워크를 정의할 수 있으므로 번거롭게 네트워크를 정의할 필요가 없습니다.
  하지만 이번에는 새로운 것을 배울 때 작고 개별적인 단계를 선호합니다.
  그렇기 때문에 사용자 지정 이름이 지정된 도커 컴포즈 파일을 사용하면 쉽게 분리할 수 있습니다.

  *추가 정보2:*</br>
  튜토리얼에서도 traefik.yml에 대한 언급이 없는 것을 볼 수 있습니다.
  그리고 traefik의 명령(command)이나 레이블(label)을 사용하여 도커-컴포즈에서 물건을 전달합니다.</br>
  [여기서](https://docs.traefik.io/getting-started/quick-start/) 처럼요. : `command: --api.insecure=true --providers.docker`</br>
  하지만 이렇게 하면 파일 구성이 조금 더 지저분해 보이고 거기서 모든 작업을 수행할 수 없습니다,
  여전히 가끔은 그 빌어먹을 traefik.yml.</br>이 필요합니다.
  그래서... 지금은 가독성이 좋은 멋진 구조의 traefik.yml을 사용하기로 했습니다.


- **traefik이 라우팅해야 하는 컨테이너에 레이블(label) 추가**.</br>
  다음은 whoami, nginx, 아파치, 포테이너의 예시입니다.</br>

  > \- "traefik.enable=true"

  traefik 활성화

  > \- "traefik.http.routers.whoami.entrypoints=web"

  진입점 웹(포트 80)에서 수신하는 `whoami`라는 라우터를 정의합니다.

  > \- "traefik.http.routers.whoami.rule=Host(`whoami.$MY_DOMAIN`)"

  이 `whoami` 라우터에 대한 규칙을 정의하는데, 구체적으로는 url 이 `whoami.whateverblablabla.org`(도메인 이름은 `.env` 파일에서 나옴)과 같을 때의 규칙입니다.
  라우터가 작업을 수행하고, 서비스로 라우팅하는 것을 의미합니다. (도메인으로 간 다음, 도메인에 연결된 서비스로 라우팅을 해준다는 말)
  
다른 것은 필요하지 않습니다. traefik은  도커 컨테이너의 컨텍스트에서 제공되는 이러한 레이블을 통해 나머지 할 일을 알고 있습니다.

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


- **그 망할 컨테이너를 실행하세요**</br>
  고아 얘기는 무시하세요. 왜냐하면 이 작성(compose) 파일들이 같은 디렉토리에 있기 때문입니다.
  그리고 작성(compose)은 작성 프로젝트의 이름으로 상위 디렉터리 이름을 사용합니다.

    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>
    `docker-compose -f apache-docker-compose.yml up -d`</br>
    `docker-compose -f portainer-docker-compose.yml up -d`


  *추가 정보 :*</br>
  작동중인 모든 컨테이너의 실행을 중지하려면 : `docker stop $(docker ps -q)`

# #2 로컬 IP 주소로 traefik을 라우팅


URL이 도커 컨테이너가 아닌 다른 것을 목표로 해야 하는 경우.


![simple-network-diagram-pic](https://i.imgur.com/lTpUvWJ.png)


- **파일 공급자 정의, 필수 라우팅 및 서비스 추가**
  
  필요한 것은 일부 URL을 포착하여 일부 IP로 라우팅하는 라우터입니다.</br>
  이전 예제에서는 포트 80에서 어떤 URL을 잡는 방법을 보여주었습니다,
  하지만 아무도 규칙에 맞는 일이 발생했을 때 어떻게 해야 하는지 알려주지 않았습니다.
  Traefik은 컨테이너의 컨텍스트에서 레이블을 사용하여 모든 작업을 수행했기 때문에 알고 있었습니다.
  `traefik.yml`에서 도커를 공급자로 설정한 덕분입니다.</br
  이 '특정 IP에서 traefik 전송'을 위해서는 Traefik 서비스가 필요합니다,
  트래픽 서비스를 정의하려면 파일 제공자, 즉 트래픽이 해야 할 일을 알려주는 빌어먹을 파일 제공자가 필요합니다.< /br>
  일반적으로는 `traefik.yml` 자체를 파일 공급자로 설정하는 것입니다.</br>
  공급자 아래에는 새로운 `파일` 섹션과 `traefik.yml` 자체가 설정됩니다.</br>
  그런 다음 동적 구성 항목이 추가됩니다.</br>
  간단한 하위 도메인 호스트 이름 규칙이 있는 `route-to-local-ip`라는 라우터입니다.
  이 규칙에 맞는 정확한 URL `test.whateverblablabla.org`은 로드밸런서 서비스로 전송되어 특정 IP와 특정 포트로만 라우팅합니다.

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
  그렇지 않으면 호스트 규칙의 도메인 이름과 해당 IP가 정의된 변수의 값을 사용합니다.
  따라서 변경하는 것을 잊어버릴 수 있으니 주의하세요.

- **traefik-docker-compose 실행**하고 작동하는지 테스트합니다.

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    
# #3 미들웨어

모든 컨테이너에 대한 인증 미들웨어의 예입니다.

![logic-pic](https://i.imgur.com/QkfPYel.png)

- **새 파일 만들기 - `users_credentials`** 사용자명:비밀번호 쌍을 포함하는,
[htpasswd](https://www.htaccesstools.com/htpasswd-generator/) style</br>
  아래 예시는 3개의 계정 모두에 비밀번호 `krakatoa`가 설정되어 있습니다.

    `users_credentials`
    ```
    me:$apr1$L0RIz/oA$Fr7c.2.6R1JXIhCiUI1JF0
    admin:$apr1$ELgBQZx3$BFx7a9RIxh1Z0kiJG0juE/
    bastard:$apr1$gvhkVK.x$5rxoW.wkw1inm9ZIfB0zs1
    ```

- **traefik-docker-compose.yml 에서 users_credentials 마운트 하기**

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

- 인증이 있어야 하는 **어던 컨테이너에다가 두 개의 레이블** 을 추가합니다</br>
  - 첫 번째 레이블은 `auth-middlewar`라는 새 미들웨어를 이미 존재하는 `whoami` 라우터에 연결합니다.
  - 두 번째 레이블은 이 미들웨어 유형은 basicauth,을 부여하고 사용자 인증에 사용할 파일이 어디에 있는지 알려줍니다.


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

- **이 망할 컨테이너를 실행**하려면 이제 로그인과 비밀번호가 필요합니다.

    `docker-compose -f traefik-docker-compose.yml up -d`</br>
    `docker-compose -f whoami-docker-compose.yml up -d`</br>
    `docker-compose -f nginx-docker-compose.yml up -d`</br>

# #4 렛츠 암호화 인증서 HTTP 챌린지 하기

![letsencrypt-http-challenge-pic](https://i.imgur.com/yTshxC9.png)

  내 프로세스에 대한 이해가 단순해졌습니다.

  `LE` - Let's Encrypt. 무료 인증서를 제공하는 서비스</br>
  `인증서` - 서버의 파일에 저장된 암호화 키입니다, 암호화 통신을 허용하고 신원을 확인합니다< /br> .
  `ACME` - 인증서를 협상하기 위한 프로토콜(정확하게 합의된 통신 방식)
LE 에서. traefik의 일부입니다.</br>
  `DNS` - 인터넷의 서버, 도메인 이름을 IP 주소로 변환</br>

  Traefik은 ACME를 사용하여 `whateverblablabla.org`와 같은 특정 도메인에 대한 인증서를 LE에 요청합니다.
  LE는 Traefik이 서버의 특정 위치에 무작위로 생성한 텍스트로 응답합니다.
  그런 다음 LE는 DNS 인터넷 서버에 `whateverblablabla.org`가 어떤 IP 주소를 가리키고 있는지 요청합니다.
  LE는 포트 80/443을 통해 임의의 텍스트가 포함된 파일에서 해당 IP 주소를 찾습니다.

  인증서를 요청한 사람이 서버와 도메인을 모두 제어하고 있다는 것을 증명하는 것이므로 DNS 레코드에 대한 제어 권한이 있습니다.
  인증서가 발급되고 3개월 동안 유효하며, 30일 미만이 남으면 자동으로 갱신을 시도합니다.


이제 실제로 완료하는 방법을 알아보세요.


- **600 권한이 있는 빈 acme.json 파일을 만듭니다**

이 파일에는 인증서와 인증서에 대한 모든 정보가 저장됩니다.

  `touch acme.json && chmod 600 acme.json`

- **443 진입점 및 인증서 확인자를 TRAEFIK에 추가합니다.yml**</br>

  진입점 섹션에 웹시큐어라는 새로운 진입점이 추가되고 포트 443 인증서 확인자는 traefik에 acme 확인자를 사용하여 인증서를 가져오는 방법을 알려주는 구성 섹션입니다.

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


  이것이 acme를 위해 필요한 전부입니다.

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

- **포트 443 노출/연결 및 traefik-docker-compose.yml에 acme.json 마운트**

acme.json이 :ro -읽기 전용- 가 **아니**라는 점에 유의하세요.

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
첫 번째 장의 일반 HTTP와 비교해보십시오,
라우터의 진입점를 `web`에서 `websecure`로 변경하는 것입니다. 그리고 `lets-encr`라는 이름의 인증서 확인자를 기존 라우터에 할당합니다. 

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

# #5 클라우드플레어에서 렛츠 암호화 인증서 DNS 챌린지 하기

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


진입점 섹션에 웹시큐어(포트 443)라는 새 진입점가 추가되었습니다.
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
- 라우터의 진입점가 `web로 전환됩니다. class="ͼe">`에서 `websecure`로 전환됩니다.
<라우터에 할당된 `lets-encr`이라는 이름의 인증서 확인자
- 인증서를 받을 기본 도메인을 정의하는 레이블입니다,
여기서는 whoami.whateverblablabla.org이며, `.env` 파일에서 가져온 도메인 이름입니다.

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

# #6 http traefik을 https로 리디렉션


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

이 라우터가 수신 대기하는 진입점를 선언합니다 - 웹(포트 80) </br>

  >\- "traefik.http.routers.redirect-https.middlewares=redirect-to-https"

는 새로 생성된 리디렉션스킴 미들웨어를 이 새로 생성된 라우터에 할당합니다.


요약하자면, 포트 80에서 요청이 들어오면 해당 진입점에서 수신 대기 중인 라우터가 이를 확인합니다.
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
