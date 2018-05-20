# Ecampus Vulnerabilities
[ecampus](http://ecampus.kookmin.ac.kr) 관련 취약점을 다룬다. 

## info
* [meta 정리](./META.md)

## Client Side Vulnerabilities
### https 와 관련해 login page 의 문제점
현재는 [login page](http://ecampus.kookmin.ac.kr/login.php) 에서만 `https` 를 반드시 사용하도록 하고 있다. 이는 로그인 시에 https 만을 이용하게 하고자 하는 의도일 것이다. 그러나, 로그인 페이지가 https 로 redirect 되는 것은 http 로 페이지가 로드되었을때 client side에서의 javascript 실행에 의한 것이다. 이는 근본적으로 단지 겉으로만 안전해 보이는 눈가리고 아웅식의 조취이다. 서버에서 'http' login request 를 받아주기 때문에 사용자는 브라우저를 이용하지 않고 직접 http 요청을 보내서 로그인 할 수 있고, MITM 공격자가 있다면, spoofing 을 하면서 login page 의 redirect javascript 만 제거하고, 정상적인 page 인 것처럼 사용자에게 건네 줄 수 있다. 서버에서 http 를 https 로 redirect 하거나 `HSTS` 를 통해 브라우저가 항상 https로 접속하도록 하는 것이 필요하다.  

### https 를 강제하지 않는 것의 문제점
보안을 위해서 사이트 전체를 https 로만 접속하도록 설정하는 것이 필요하다. 예를 들어, 사이트에는 퀴즈 기능이 있는데, http 접속을 받아주면 중간자 공격에 의해 퀴즈 답변 내용이 노출 될 수 있다.

### Protocol Downgrade Attack 방지 (SSLStrip)
https 를 강제하는 것이 보안을 보장하지 않는다. 
중간자가 SSL/TLS 를 Downgrade 해서 client 에게는 plain http 를 사용하게 하고, 중간자는 그것을 받아 정보를 획득한 후, 다시 암호화해서 server에게 보내면 일반적으로  server는 이 과정을 눈치채지 못한다. `SSLStrip` 으로 대표되는 이런 공격을 방어하기 위해서는 `HSTS(Http Strict Transport Security)` 를 설정하는 것이 필요하다.
구체적으로 말해, 
```
Strict-Transport-Security: max-age=31536000
``` 
와 같은 헤더를 설정하면, 브라우저는 모든 연결을 https 로 강제한다 (31536000초=1년간). (브라우저는 http 연결을 거부한다.)  물론, client 가 처음으로 방문하는 것이라면 중간자는 HSTS 만 제거하고 내용을 건제줄 수 있지만, 이런 경우를 제외하면 훨씬 안전해진다. 물론, 주의할 점은, 몇몇 웹사이트의 경우 인증서를 제 때 갱신하지 않는데, 만일 이렇게 되면 인증서 만료시 서비스 자체가 수행되지 않을 수 있으니, 항상 인증서가 유효하다는 전제가 필요하다.   

### CSRF 취약점
패킷을 캡처하고, 소스를 분석해 본 결과, 실험해 본 모든 request 에서 `CSRF` 공격 방지를 위한 토큰이 감지되지 않았다. 학생들이 이용하는 사이트인 만큼, ecampus 외부의 학생 커뮤니티에서 ecampus로 몰래 request 를 보내는 악성 링크 또는 스크립트 삽입되거나, 또는 서로 아는 사이에서 소셜 해킹에 의해 CSRF 공격이 실행될 여지가 있다. 따라서 response 로 보내는 page 마다 일회성인 csrf prevention token 을 삽입해 공격을 방지하는 것이 필요하다.   

### 쿠키 및 세션과 관련된 문제점
#### MoodleSession 에 httpOnly flag 가 붙지 않은 문제  
ecampus 에서 세션 식별용 쿠키의 이름은 MoodleSession 이다.
MoodleSession 에 `httpOnly` flag 가 설정되어 있지 않으므로, 브라우저는 javascript 를 통해 cookie에 접속하는 것을 허용한다. 예를들어, 
```js
document.cookie;
```
위 명령어를 브라우저 console 에서 실행시 다음과 같이 모든 쿠키의 정보를 보여준다.
```js
"_ga=GA1.3.1980483938.1523699208; moodle_notice_1_53268=hide; webmail_id=20171580 _gid=GA1.3.84539261.1526729405; MoodleSession=qlcjb4i5gqtvb8jslfjimgbjv5; _gat=1"
```
이는 혹여라도 (유지보수 등을 통해 미래에 발생할 취약점을 포함해) 있을 XSS 공격이나 다른 창의적인 공격 시나리오에서 응용될 수 있기에 위험하다. 

#### secure flag 가 붙어있지 않은 문제 
https를 강제한다면, 쿠키에 `secure` flag 를 붙여, Protocol Downgrade Attack 에 대해 한번 더 안전하게 만드는 것이 필요하다.

#### SameSite가 붙어있지 않은 문제
쿠키에서 `SameSite` flag는 비교적 최근의 속성이므로, 모든 브라우저에 구현되어 있는 것은 아니다. 그러나, 브라우저는 외부 도메인의 사이트에서 request를 보낼 때, `SameSite` flag 가 붙은 쿠키를 전송하지 못하게 하여, MoodleSession 과 같이 민감한 쿠키 를 전송하지 않아 CSRF를 방지하는 데에 도움을 준다.   

### Information Leakage
[Moodle FW](https://github.com/moodle/moodle) 를 사용했다는 것을 session 명, html meta 정보 등에서 명시적으로 확인 할 수 있었다. 향후, 프레임워크에 취약점이 발견되면, 패치가 될 것인데, 아마도 ecampus에 그 패치가 적용되는 시점은 그 보다 꽤나 이후일 수가 있다. 즉, 제로데이 공격을 받을 수 있다. 때문에 불필요하게 ecampus가 moodle 을 통해 만들어졌다는 것을 표시할 이유는 없다고 여겨진다. 

## Server Side Vulnerabilities
### candidates
* IP 
* DOS
* Server 정보 

### Information Leakage
#### database 정보 노출  
  **예** : `http://ecampus.kookmin.ac.kr/pluginfile.php/-10/user/icon/` 접속시 
  ```
  데이터베이스 테이블 context 에서 데이터 레코드를 찾지 못 함
  ```
이라는 설명이 나온다. 가능한 한 데이테베이스에 관련한 raw information 이 end user 에게 노출 되지 않는 것이 안전하다. 

## 개발 관련 내용
취약점 중심으로 분석했지만, 일부 개발 관련 내용에 대한 첨언을 달아본다.

### Task Runner를 사용하는 것을 권장 
만일 `grunt`,`gulp` 등의 Task Runner를 사용하지 않는다면, 사용하는 것을 권장한다. 혹시 Task Runner를 사용하지 않는 것인지 추측한 이유는, minified 되지 않은 javascript와 css를 로딩하고, 상당히 여러개의 js 파일을 로딩하는 등을 보았기 때문이다. 만일, 이미 Task Runner를 사용한다면, 적절한 plugin 을 사용해, 해당 부분들을 효율화하면 좋겠다. 

### 인코딩은 UTF-8로
일부 소스에서 한글이 깨지는 것으로 보아 인코딩에 문제가 있다고 여겨진다.
예 : http://ecampus.kookmin.ac.kr/theme/jquery.php/theme_coursemosv2/jquery.validate.ko.js

### webpack
`require.js`를 사용하는 것으로 보이는데, 향후 효율, 유지보수, 인수인계의 용이성을 위해서 `webpack`을 사용하는 것이 어떨까?

## what seems safe
* file upload/download vulnerability
* XSS