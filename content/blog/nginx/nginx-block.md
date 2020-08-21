---
title: How Nginx blocks work
date: "2020-01-13T22:12:03.284Z"
description: "Nginx 의 block config 를 알아보자"
---

이글은 https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms 를 번역/참조 했습니다.

Nginx 의 block config를 알아보자.

Nginx는 여러개의 서버블록 설정이 가능하다. 가상의 웹서버들을 만들어 분리하고 request에 맞는 웹서버를 실행시킨다.
그리고 request를 받은 메인 서브블록은 listen 과 server_name 를 고려해서 프로세스를 진행시킨다.

그렇다면 Nginx는 어떻게 request 를 만족시키는 프로세스를 진행할까?

1.  'listen' directive 를 파싱해서 매치되는 블록을 찾는다.

    Nginx는 request 의 IP 주소와 port 를 찾는다. 그리고 각 서버의 listen directive 와 비교해서 resolve 가능한 서버 블록들을 나열한다.
    'listen' directive 에는 IP 주소와 port 가 서버에 응답 될 수 있게 정의되어 있다. 그리고 디폴트로 'listen' directive 가 없는 서버 블록은
    0.0.0.0:80 혹은 0.0.0.0:8080(Nginx 가 non-root user 로 실행되고 있을 경우) 가 주어진다. 이 방식은 port 80 의 어떤 인터페이스이든지
    request에 응답할 수 있게 한다. 하지만 디폴트 값은 이 서버 셀렉션 프로세스에 큰 비중을 차지 하지 않는다.

    'listen' directive 는 다음과 같이 설정될 수 있다.

        - IP address/port 조합

        - default port 80 을 바라보는 IP address

        - 모든 인터페이스를 바라보게 하는 포트

        - Unix socket 경로

    Nginx 가 어떤 서버 블록으로 request 를 보낼지는 구분할 때, 첫번째로 'listen' directive 을 기반으로 다음과 같은 rule 에 의해 결정한다.

        - 'listen' directive 에 완전하지 못한 부분들을 default values 로
        채우며 translate 를 한다.

    그리고 각각의 블록을 IP 와 port 로 평가 되게 한다. 예를 들자면 다음과 같다.

        - 'listen' directive 에 아무것도 없는 블록은 0.0.0.0:80 으로 채워진다.

        - port 없는 111.111.111.111 는 111.111.111.111:80 으로 채워진다.

        - port 8888 에 IP address 가 없으면 0.0.0.0:8888 으로 채워진다.

        - 그다음, IP address 와 port 를 기반으로 request와 가장 명확하게 맞는
        서버블록 리스트를 모은다. 이말은 만약에 명확한 IP address 가 있는 블록이
        있다면 IP address 가 0.0.0.0 인 블록은 선택 받지 못하게 된다.
        어떤 경우라도 port 는 정확하게 매치가 되야한다.

        - 가장 명확하게 매치가 되는 서버 블록이 1개 있다면 그 블록이
        request 에 응답한다. 만약 비슷한 정확도로 매칭되는 서브블록이 여러개라면
        Nginx 는 'server_name' directive 를 평가하게 된다.

    'listen' directive 에 같은 정확도의 서버 블록이 여러개가 있다면, Nginx 는 블록의 'server_name' 을 찾아보게 된다.
    예를 들어 example.com 가 port 80 에 192.168.1.10 로 호스트 되면, example.com 의 request 는 항상 첫번째 블록에서 serve 된다.

        server {
            listen 192.168.1.10;
            . . .
            }

        server {
            listen 80;
            server_name example.com;
            . . .
            }

    이제 Nginx 는 가장 정확한 매칭을 찾기 위해 'server_name' directive 를 아래와 같은 rule 으로 보게된다.

        - 첫번째로 'server_name' directive 의 값이 request 의
        'Host' header 와 정확하게 맞는지 확인한다. 여러 블록이 매칭된다면
        첫번째 블록을 사용한다.

        - 매치 되는 블록을 찾을 수 없으면, 'leading wildcard'
        (indicated by a * at the beginning of the name in the config)
        를 사용하는데 1개의 블록을 찾게되면 그 블록을 사용하고 여러 블록을 찾게 되면
        그 중에 가장 길게 매칭되는 블록을 사용한다.

        - 다음은 regular expression(indicated by a * at the
        beginning of the name in the config)
        을 이용해서 'Host' header 와 매치되는 첫번째 블록을 사용한다.

        - 그래도 매치되는게 없으면, default 서버 블록을 선택한다.

    각 IP address/port combo 는 default 서버 블록을 가지고 있고 위 방법들을 통해 적합한 블록을 찾아
    실행할 수 없게 되면 첫번째 서버 블록이나 'default_server' 옵션을 가지고 있는 블록을 실행한다.
    'default_server' 가 선언된 IP address/port combination 은 단 한 개 밖에 없다.

그 다음은 Location Block 에 대해 알아보자.

Location block 은 server block 에 있는데 domain name 이나 IP address/port 뒤에 오는 request URI 를
어떻게 실행 시킬지를 결정한다.

Location blocks 는 일반적으로 아래와 같은 형태로 되어있다.

    location optional_modifier location_match {

    . . .

    }

'location_match' 는 Nginx 가 request URI 에서 어떤 것을 확인해야하는지 정의해준다.
modifier 가 있고 없음에 따라 Nginx 가 location block 을 매칭시켜주는게 달라진다. 아래 modifiers 는
location block 이 어떻게 해석되어지는지를 보여준다.

    - modifier 가 없으면 location 은 prefix 로 해석되어 매칭된다.
    그 말은 주어진 location 은 request URI 의 앞에서부터 매칭 분석이 진행된다.

    - '=' 을 사용하면 request URI 가 주어진 location 과
    정확히 매치가 되어야 된다.

    - '~' 이 사용되면
