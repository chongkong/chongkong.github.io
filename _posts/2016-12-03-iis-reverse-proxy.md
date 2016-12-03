---
layout: post
title: "IIS에서 Reverese Proxy 설정하기"
date: 2016-12-03 21:27:29 +0900
categories: iis
tags: ['iis', 'reverse proxy', 'arr', 'application request routing', 'url rewrite']
comments: true
---

Slack에 [interactive button](https://api.slack.com/docs/message-buttons)을 추가하기 위해서는 
Slack에서 버튼 클릭 이벤트를 전달할 수 있는 endpoint가 필요하다. 이를 request_url이라 하는데 
반드시 https url을 주어야 한다.

![enable-interactive-message](assets/posts/iis-reverse-proxy/enable-interactive-message.png)

로컬에서 서버를 서빙하며 ssl 인증서를 사용하기 위해서는 apache나 nginx 등의 웹서버를 이용하는 편이 좋은데
윈도우에서는 IIS를 이용해서 할 수 있다. 이번 포스트에서는 IIS를 이용하여 SSL 인증서를 서빙하고
nginx나 apache의 virtual host 기능을 포함한 reverse proxy를 어떻게 설정하는지 공유해보고자 한다.

<div style="background-color: rgba(255, 96, 96, 0.2); border-radius: 4px; border: 1px solid rgba(255, 96, 96, 0.8); padding: 10px">
<strong>Disclamier</strong><br>
이 글은 IIS 홈페이지에서 제공하는 <a href="https://www.iis.net/learn/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing">문서</a>
를 최신 IIS (10.0) 버전에 맞게 + 읽기 쉽게 각색한 글입니다.
</div> 

# 준비하기

나는 가난한 학생이라 wildcard SSL 인증서를 살 돈이 없다 😂 Namecheap 에서 제공하는 
[wildcard 인증서](https://www.namecheap.com/security/ssl-certificates/wildcard.aspx) 가격을 보면
제일 싼 인증서가 연간 $94.00에서 시작한다. 3개 도메인은 연간 $29.88, 싱글 도메인은 연간 $9.00 이다.

이 글에서는 싱글 도메인 인증서를 기준으로 프록시를 설정할 것이다. 다시 말해

- `https://foo.yourdomain.com` -> `http://localhost:8001`
- `https://bar.yourdomain.com` -> `http://localhost:8002`

와 같은 서브도메인을 이용한 프록시 설정은 여러개의 도메인에 대한 인증서가 필요하므로

- `https://yourdomain.com/foo` -> `http://localhost:8001`
- `https://yourdomain.com/bar` -> `http://localhost:8002`

와 같이 path를 나눠먹는 프록시를 설정하고자 한다. 사실 설정 방법에 큰 차이가 있는 것은 아니라, 서브도메인을
이용한 프록시가 어떤 값을 다르게 하면 되는지는 해당 부분에서 언급하겠다.

## 기본 준비물

- 내 도메인
- SSL 인증서
- IIS가 enable된 윈도우

## IIS에 인증서 등록하기

프록시를 설정하며 꼭 HTTPS를 사용할 필요는 없다. 이미 별도로 인증서가 붙은 로드밸런서로부터 트래픽을 받는 상황이라
HTTP만으로 프록시를 구성하고 이 단계를 건너뛰면 된다. 하지만 단지 귀찮아서 HTTPS를 사용하지 않는 것이라면 이 기회에 인증서를
구입해서 HTTPS를 지원하도록 하는 편이 좋다. 특히 하나의 머신을 사용하고 있다면 (보통 자신의 집컴이겠지만) IIS와 같은 웹서버에
인증서를 붙이는 것이 일반적이다.

# 프록시 설정하기

프록시의 기본적인 흐름은 다음과 같다.

![basic-flow](assets/posts/iis-reverse-proxy/basic-flow.png)

## IIS에 사이트 만들기

사이트는 IIS가 서빙하는 단위이다. 하나의 사이트는 하나 이상의 바인딩을 포함할 수 있다.

![create-site](assets/posts/iis-reverse-proxy/create-site.png)

바인딩에서는 호스트 이름을 선택할 수 있다. 하나의 머신에서 여러 개의 사이트를 운용하려면 외부에서 머신의 IP로 접속했을 때 각 사이트를 
구분할 방법이 필요한데 이 때 호스트 이름을 사용한다. 즉 같은 IP 엔드포인트를 가지더라도 `foo.com`으로 접속했을 때와 `bar.com`으로 접속했을 때 
다른 사이트로 라우팅이 되게 할 수 있다.

하지만 서브도메인을 이용해서 라우팅을 할 때에는 여러 개의 사이트를 이용하기보다는 하나의 사이트에 wildcard를 이용한 바인딩을
설정해야 한다. 바인딩의 호스트 이름으로 `*.yourdomain.com`를 사용하면 `yourdomain.com`의 모든 서브도메인을 지칭하는 것이다.
(내가 가진 SSL 인증서는 싱글 도메인이므로 호스트 이름으로 `yourdomain.com`을 사용했다.) 또한 바인딩 종류에서 https를 선택하면 
IIS에 등록된 인증서 중 하나를 고를 수 있는데 해당 primary domain의 와일드카드 인증서를 고르면 딱 된다. 

한편 사이트는 반드시 컨텐츠 디렉터리를 설정해야 한다. 컨텐츠 디렉터리는 정적 파일들을
서빙할 호스트의 경로를 의미하는데, 우리는 별도로 서빙할 파일이 없으니 적당한 빈 디렉터리를 가리키게 한다. (기본 경로는 `C:\inetpub\wwwroot`에 있다)

## ARR과 URL Rewrite 설치하기

프록시를 설정하기 위해서는 [Application Request Routing 모듈](https://www.microsoft.com/en-us/download/details.aspx?id=47333)(이하 ARR)과
[URL Rewrite 모듈](https://www.microsoft.com/en-us/download/details.aspx?id=47337)을 설치해야 한다.

### ARR

ARR은 바인딩을 통해 들어온 요청을 프록시하기 위해 필요하다. ISS 설정에서 Application Request Routing 항목이 생겼을 것이다.

![arr-setting-1](assets/posts/iis-reverse-proxy/arr-setting-1.png)

해당 메뉴에서 서버 프록시 세팅 항목으로 들어간다.

![arr-setting-2](assets/posts/iis-reverse-proxy/arr-setting-2.png)

여기서 `Enable proxy`를 체크해주자. 또 프록시된 요청은 일정 시간동안 캐싱이 되는데 캐싱이 필요하지 않은 경우엔 
`Cache Setting`의 `Memory cache duration`을 0초로 해 두자.

![arr-setting-3](assets/posts/iis-reverse-proxy/arr-setting-3.png)

한편 맨 아래에 `Proxy Type`에서 `Use URL Rewrite to inspect incoming requests` 항목이 있을텐데 
세부 프록시 설정은 ARR이 아닌 URL Rewrite 메뉴에서 별도로 설정해 줄 것이기 때문에 건드리지 말자. 

### URL Rewrite

이제 프록시 기능이 활성화되었으므로 특정 바인딩에 대하여 URL Rewrite를 통해 다른 엔드포인트로 라우팅을 해줄 수 있다.

먼저 아까 만든 사이트의 IIS 설정에서 URL Redirect 메뉴를 들어가 보자.

![url-rewrite-setting-1](assets/posts/iis-reverse-proxy/url-rewrite-setting-1.png)

우측 상단에 Add Rule을 선택하면 다음과 같이 Reverse Proxy 항목이 있다. 만약 Reverse Proxy 항목이 보이지 않는다면
사이트의 IIS 설정이 아니라 전체 홈의 IIS 설정을 들어간 것이니 사이트 설정 페이지에 들어간 뒤에 다시 URL Rewrite 메뉴로 들어간다.

![url-rewrite-setting-2](assets/posts/iis-reverse-proxy/url-rewrite-setting-2.png)

그리고 다음과 같이 설정해 준다. 만약 서브도메인을 사용해서 프록시를 할 것이라면 `yourdomain.com` 대신 `foo.yourdomain.com`을 입력해 주면 된다.
`yourdomain.com/foo`와 같이 프록시를 하는 경우에는 일단 호스트 이름으로는 `yourdomain.com`을 입력해 주자.

![url-rewrite-setting-3](assets/posts/iis-reverse-proxy/url-rewrite-setting-3.png)

그러면 Inbound Rule과 Outbound Rule이 각각 하나씩 생길 것이다. subpath를 이용하여 프록시를 하는 경우 Outbound Rule에 path가 지정되지 않았으므로
직접 Outbound Rule을 수정하여

```
http{R:1}://yourdomain.com/{R:2}
```

으로 되어있는 부분을

```
http{R:1}://yourdomain.com/foo/{R:2}
``` 

로 바꿔주면 된다. `{R:1}`과 `{R:2}`는 각각 Pattern에서 정규식의 첫 번째 그룹과 두 번째 그룹을 의미한다.

모든 설정이 끝났으면 `yourdomain.com` 사이트를 재시작 해주자.

# Trouble Shooting

이제 프록시 설정은 완료되었지만 접속이 안 된다면 다음 사항들을 하나씩 확인해 보자.

1. 서브도메인 프록시를 사용중이라면 모든 서브도메인들에 대해 IP주소를 모두 지정해 두었는가?
2. 윈도우 방화벽에서 World Wide Web 서비스 HTTPS 인바운드를 허용했는가? 
   (윈도우 방화벽에서 HTTPS 인바운드 규칙은 이미 정의되어 있지만 기본으로 활성화되어있지 않기 때문에 꼭 활성화를 해 주어야 한다.)
3. 공유기를 사용중이라면 포트포워딩 설정을 했는가?

------

### Reference

- [Reverse Proxy with URL Redirect v2 and Application Request Routing](https://www.iis.net/learn/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing)
