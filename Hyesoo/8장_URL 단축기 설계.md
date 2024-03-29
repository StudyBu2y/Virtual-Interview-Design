# 8장 URL 단축기 설계

## 1단계 요구사항 분석

- 매일 1억개의 단축 URL 생성
- 초당 1160(100million/24/3600)
- 읽기연산과 쓰기연산의 비율을 10:1이라고 할때 읽기연산은 초당 11,600회
- URL 단축 서비스를 10년간 운영한다고 가정하면 3650억 개의 레코드를 보관해야함
- 축약 전 URL의 평균 길이를 100이라고 하면 10년 동안 필요한 저장 용량은 3650억 * 100 바이트 → 36.5TB

## 2단계 개략적설계안 제시 및 동의 구하기

### API엔드포인트

REST스타일 설계 하에서 URL단축기는 기본적으로 두 개의 엔드포인트를 필요로 한다.

- URL 단축용 엔드포인트
    - 새 단축 URL을 생성하고자 하는 클라이언트는 이 엔드포인트에 단축할 URL을 인자로 실어서 POST요청을 보내야한다.  *쉽게 말해 단축 URL을 얻는 API를 요청한다는 것*
    - POST `/api/v1/data/shorten`
        - parameter : {longUrl: longUrlString}
        - return : 단축 url
- URL 리디렉션용 엔드포인트
    - 단축 URL에 대해서 http 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트
    - GET `/api/v1/shortUrl`
        - 반환 : HTTP 리디렉션 목적지가 될 원래 URL

### URL 리디렉션

![스크린샷 2024-02-29 10.58.08.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_10.58.08.png)

![스크린샷 2024-02-29 10.58.21.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_10.58.21.png)

- 응답 상태 코드의 차이
    - 301 permanenetly moved → 요청에 대한 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답
        - 영구적으로 이전되었기 때문에 브라우저는 응답을 캐시 처리하여 재 요청이 들어왔을 경우 캐시된 api를 참조한다.
    - 302 Found → 주어진 URL로의 요청이 일시적으로 Location 헤더에 반환된 URL에 의해 처리되어야 한다는 응답
        - 캐시하지 않고 항상 단축 URL 서버에 요청해야함.
    - 서버 부하를 줄이기 위해서는 301, 트래픽 분석이 중요할 때는 302를 사용하자
- URL 리디렉션을 구현하는 가장 직관적인 방법은 해시 테이블을 사용하는 것
    - 원래 URL = hashTable.get(shorten URL)
    - 301, 302 reposnse 와 URL을 header 담아 전송

### URL 단축

- **해시함수 요구사항**
    - 입력으로 주어지는 URL이 다른 값이면 해시 값도 달라야한다.
    - 게산된 해시 값은 원래 입력을 주어졌던 긴 URL로 복원가능해야한다.

## 3단계 상세 설계

### 데이터 모델

모든 것을 해시테이블에 두는 설계는 단순하지만 메모리 이슈가 있어 실제 시스템에 쓰기는 어렵다. 더 나은 방법은 Pair<shorten URL, long URL>을 DB에 저장하는 것이다.

### 해시함수

- 해시값 길이
    - hashValue는 [0-9, a-z, A-Z]의 문자로 구성 → 62개로 구성
    - 62^n > 3650억(10년간 보관할 단축 url 개수)을 만족하는 n의 최솟값을 찾아야함 **(몇 비트로 구성할지)**
    - n = 7
- 해시 후 충돌 해소
    - 긴 URL을 7글자 문자열로 줄이는 가장 손쉬운 방법은 잘 알려진 해시 함수 이용하기(SHA-1, MD5 등)
    - 그러나 가장 짧은 해시결과값도 7보다 길다 ! 해결법 ?
        - 계산된 해시 값에서 처음 7개 글자만 이용하는 것 *→ 충돌 확률 높아짐*  → 충돌 해소될때까지 사전에 정한 문자열을 해시값에 덧붙임
            
            ![스크린샷 2024-02-29 11.19.06.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_11.19.06.png)
            
        - 충돌 해소는 되지만 쿼리가 많아지는 오버헤드가 있음 → **블룸 필터 사용하여 성능 개선**
- base62 변환
    - 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우 유용함
    - 유일 id를 shortenURL로 변환시켜 저장하는 것
    - 62개의 문자를 사용하기 때문에 표현할 수 있는 62개를 적용한 62진법 사용
    - 10진수로 11157을 62진법으로 변환 → 2TX

![스크린샷 2024-02-29 11.22.51.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_11.22.51.png)

### URL 단축기 상세 설계

- 순서도
    
    ![스크린샷 2024-02-29 11.23.59.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_11.23.59.png)
    
- DB 스키마
    
    
    | ID | shortURL | longURL |
    | --- | --- | --- |
    | 2009215674938 | zn9edcu | https://en.wikipedia.org/ wiki/Systems_design |
- 상세 설계
    
    ![스크린샷 2024-02-29 11.27.10.png](8%E1%84%8C%E1%85%A1%E1%86%BC%20URL%20%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8E%E1%85%AE%E1%86%A8%E1%84%80%E1%85%B5%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A8%2071d568eeee7544bd8acfcccfe8b69a1c/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2024-02-29_11.27.10.png)
    
    ## 4단계 마무리
    
    - 처리율 제한 장치를 두어 URL단축기에 대한 트래픽에 대비할 수 있다
    - HTTP 무상태 계층이므로 웹 서버의 규모 확장 용이
    - 데이터 베이스 규모 확장(다중화, 샤딩)
    - URL단축 서버에 분석기를 달아 모니터링 가능