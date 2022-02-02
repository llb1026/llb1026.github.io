---
order: 8
layout: post
title: "혼자 해 본 백엔드 성능테스트"
subtitle: "백엔드 최적화 포인트와 nGrinder를 이용한 성능테스트 결과"
tag: Lesson Learned
type: lesson-learned
blog: true
text: true
author: JIYUN LEE
post-header: true
header-img: img/01_1.png
next-link: "../naver-intern-epilogue/"
prev-link: "../client-side-routing/"
---

## 1. 백엔드 성능 테스트

성능 테스트란

- 오픈할 시스템의 병목을 찾고,
- 시스템의 성능 기준을 잡고,
- 성능 튜닝의 대상을 찾는데 도움이 되고,
- 성능 목표와 관련된 요구사항을 찾고,
- 성능과 관련된 데이터들을 모으고,
- 하드웨어의 최적화된 설정을 잡는 작업이다.

### 1-1. 성능 테스트의 주요 관점

- **응답시간 측정**
    - 응답시간은 사용자의 이탈과 밀접한 관련이 있음 → 가장 중요!
    - 부하시 응답시간이 어떻게 변화하는지 확인
- **처리량**
    - 단위 시간당 처리할 수 있는 양을 확인
    - 일반적으로는 초 단위인 **TPS**(Transaction Per Seconds)를 측정
        - Transaction을 잡는 기준은 달라질 수 있음 (단순히 하나의 요청에 대한 응답으로 잡을 수도 있고, 규모가 큰 프로젝트인 경우 하나의 비즈니스 로직을 트랜잭션 하나로 잡을 수도 있음)
    - 부하시 처리량이 어떻게 변화하는지 확인
- **확장성**
    - 서버를 추가해야 하는 상황이 발생할 것을 대비해 어떤 서버를 얼마나 추가해야 할지에 대한 사전 확인
- **부하시 오류 발생 여부 확인**
    - 로컬 개발환경에서는 발생하지 않는 오류가 운영 및 부하 상황에서 발생할 확률이 매우 높음
- **병목지점 도출과 튜닝을 위한 정보 제공**
    - 응답시간 및 처리량에 영향을 주는 병목 지점을 찾아 사전에 제거
    - 서버의 설정이 최적화되어 있는지에 대한 점검
    - Lock, Resource contention 발생 여부 확인 가능

### 1-2. 서버 성능 테스트 시 제외 대상

- Front End (Client)의 성능 제외
    - 브라우저에서 script 처리하고 화면 레이아웃 구성하는 시간은 별도의 측정 툴로 봐야 함 → NSPEED
- 유저의 네트워크 구간 제외
    - 사용자의 요청이 IDC 네트워크 망 안으로 들어왔을 때부터의 시간만 측정 가능함
    - 예를 들어 사용자가 모뎀이나 3G 망을 써서 IDC 네트워크 망으로 들어올 때까지의 시간 측정은 불가능

### 1-3. 테스트 절차

1. 계획
    - 성능 테스트의 대상과 범위를 결정
2. 준비
    - 부하를 가하는 스크립트 준비
    - 대상 시스템을 production 모드로 변경
    - 각종 리소스 모니터링 방안 마련
3. 실시
    - 시스템을 모니터링하면서 테스트 실시
    - 병목지점을 발견하거나 문제가 발생하면 테스트를 멈추고 원인 파악 및 해결
4. 결과 분석 및 정리
    - 분석한 결과에 문제가 없으면 결과를 정리
    - 문제가 있으면 튜닝 및 문제점을 보완한 후 테스트 재실시
5. 결과 공유
    - 서비스의 오픈과 관련된 담당자들에게 결과 공유

### 1-4. 관련 용어

- *Agent*: Active 상태인 부하 발생기의 수
- *Process*: Agent당 부하를 발생시키는 Process의 개수
- *Thread*: Process당 사용되는 부하 발생 Thread의 개수
- *Target Host*: 모니터링 대상의 TP
- *Simple Interval*: 실시간 모니터링 그래프를 나타내기 위한 데이터 수집 간격
- *Ignore*: 테스트 수행 시작 직후 warm-up을 위해 데이터 수집을 무시할 초(sec) 값
- *Expect Vuser*: 사용자가 생성하고자 하는 Virtual User 수
- *Actual Vuser*: Agent, Process, Thread에 의해 생성되는 실제 Vuser 수
- *Ran Counts*: 각 테스트가 수행되는 횟수 (0으로 설정하면 Duration에서 설정한 시간만큼 수행)
- *Duration*: 테스트 수행 시간 (Run Count가 1 이상이면 Run Count 값이 우선됨)

### 1-5. 관련 공식

- Little's Law
    - Concurrent User = TPS * (Avg. Response Time) + Think Time
- Amdahl's Law
    - 1 / {(1-P) + P/S}, (P = 전체 시스템에서 차지하는 비중, S = 성능 향상 비율)
    - 개선 후 실행 시간 = 개선에 의해 영향을 받는 실행 시간 / (성능 향상 비율 + 영향을 받지 않는 실행 시간)
    - 예를 들어, 전체 시스템 중 40%에 해당하는 부분의 속도를 2배로 늘렸을 경우 예상되는 최대 성능 향상은 1 / {(1-0.4) + 0.4/2} = 1.25

---

## 2. 최적화 포인트

### 2-1. 비즈니스 로직

추후 추가 예정

### 2-2. 쿼리

추후 추가 예정

### 2-3. 웹 서버 세팅

#### 2-3-1. keepalive

Nginx upstream을 설정할 때 다음과 같이 Nginx와 WAS간의 세션을 유지하도록 keepalive를 켤 수 있다.

```conf
upstream backend {
    server example.com:8080;
    keepalive 100;    // keepalive로 유지시키는 최대 커넥션 개수
}
```

만약 keepalive가 켜져 있지 않다면 리퀘스트마다 새로운 세션을 만들어 한 번의 요청만 처리한 후 소켓이 끊어진다. 이로 인해 시간이 지날수록 불필요한 TCP handshake가 발생해 응답 속도 지연을 일으킬 수 있다.

#### 2-3-2. TCP keepalive vs Nginx keepalive

헷갈릴 수 있어서 같이 정리한다.

*TCP* keepalive와 *Nginx* keepalive 모두 TCP 세션을 유지하기 위해 사용하지만, 다음과 같은 큰 차이점들이 있다.

1. TCP keepalive는 keepalive 유지 및 관리에 대한 작업을 커널이 직접 한다. TCP 계층에서 직접 해당 소켓이 살아있는지 확인하는 반면, Nginx keepalive는 기본적으로 어플리케이션 레벨에서의 keepalive이다.
2. TCP keepalive는 주기적으로 ping-pong을 통해 상대방이 살아있는지 확인해 일정 횟수동안 응답이 없으면 연결을 끊지만, Nginx keepalive는 클라이언트로부터 먼저 FIN을 받지 않는 이상 먼저 연결을 끊지 않고 keepalive time동안 세션을 유지한다 → 반대로 다시 표현하자면, TCP keepalive는 ping-pong에 대한 응답이 온다면 끊지 않고 계속해서 세션을 유지하지만, Nginx keepalive는 설정된 keepalive time이 지나면 클라이언트와의 연결을 능동적으로 끊는다.

둘은 비슷해 보이지만 서로 연관성이 없다. 만약 어플리케이션에서 자체적으로 keepalive 세션을 관리한다면 TCP keepalive 값은 무시해도 된다. ([참고](https://brunch.co.kr/@alden/9))

#### 2-3-3. timeout 설정시 고려할 점

적절하게 설정되지 않은 keepalive는 웹 서버의 스레드 부족 현상을 일으킬 수 있다. 

예를 들어, 1000개의 worker가 떠 있는 웹 서버에 1000개의 세션이 keepalive를 켜고 들어오면 1001번째 세션은 처리를 받아줄 worker가 없기 때문에 스레드가 부족해지고, 서비스의 전체적인 응답속도가 저하된다.

따라서 서비스의 트래픽과 요청량에 따라 적절한 값의 keepalive timeout을 설정해야 한다. 

---

## 3. 테스트 환경

- 기사 수: 12개
- 에이전트: 3
- 에이전트 별 Vuser: 150
- 총 Vuser: 450
- 프로세스: 4
- 스레드: 37
- 테스트 기간: 1시간
- 초기 대기시간: 1000ms
- 테스트 스크립트:

```java
import static net.grinder.script.Grinder.grinder
import static org.junit.Assert.*
import static org.hamcrest.Matchers.*
import net.grinder.plugin.http.HTTPRequest
import net.grinder.plugin.http.HTTPPluginControl
import net.grinder.script.GTest
import net.grinder.script.Grinder
import net.grinder.scriptengine.groovy.junit.GrinderRunner
import net.grinder.scriptengine.groovy.junit.annotation.BeforeProcess
import net.grinder.scriptengine.groovy.junit.annotation.BeforeThread
// import static net.grinder.util.GrinderUtils.* // You can use this if you're using nGrinder after 3.2.3
import org.junit.Before
import org.junit.BeforeClass
import org.junit.Test
import org.junit.runner.RunWith

import java.util.Date
import java.util.List
import java.util.ArrayList

import HTTPClient.Cookie
import HTTPClient.CookieModule
import HTTPClient.HTTPResponse
import HTTPClient.NVPair

/**
 * A simple example using the HTTP plugin that shows the retrieval of a
 * single page via HTTP. 
 * 
 * This script is automatically generated by ngrinder.
 * 
 * @author 이지윤
 */
@RunWith(GrinderRunner)
class TestRunner {

	public static GTest test
	public static GTest testSection
	public static GTest testArticle
	public static HTTPRequest request
	public static NVPair[] headers = []
	public static NVPair[] params = []
	public static Cookie[] cookies = []
    public static String HOST = ""

	@BeforeProcess
	public static void beforeProcess() {
		HTTPPluginControl.getConnectionDefaults().timeout = 6000
		test = new GTest(1, HOST)
		testSection = new GTest(1, HOST)
		testArticle = new GTest(1, HOST)
		request = new HTTPRequest()
		grinder.logger.info("before process.");
	}

	@BeforeThread 
	public void beforeThread() {
		test.record(this, "test")
		testSection.record(request)
		testArticle.record(request)
		grinder.statistics.delayReports=true;
		grinder.logger.info("before thread.");
	}
	
	@Before
	public void before() {
		request.setHeaders(headers)
		cookies.each { CookieModule.addCookie(it, HTTPPluginControl.getThreadHTTPClientContext()) }
		grinder.logger.info("before thread. init headers and cookies");
	}

	@Test
	public void test(){
		HTTPResponse result = request.GET(HOST, params)

		if (result.statusCode == 301 || result.statusCode == 302) {
			grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", result.statusCode); 
		} else {
			assertThat(result.statusCode, is(200));
		}
	}
	
	@Test
	public void testSection() {
		// 섹션별 기사 목록
		String[] section = ["news", "entertain", "sports"];
		for (int i = 0; i < section.length; i++) {
			HTTPResponse result = request.GET(HOST + section[i] + "/0", params);
			
			if (result.statusCode == 301 || result.statusCode == 302) {
				grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", result.statusCode);
			} else {
				assertThat(result.statusCode, is(200));
			}
		}
	}
	
	@Test
	public void testArticle() {
		// 기사 상세보기
		String[] oid = ["011", "055", "015", "052", "003", "311", "213", "112", "011", "018", "047", "436"];
		String[] aid = ["0003461916", "0000695140", "0004059512", "0001227035", "0008954639", "0000931431", "0001075192", "0003107268", "0003461852", "0004269678", "0002211348", "0000030345"];
		
		for (int i = 0; i < oid.length; i++) {
			HTTPResponse result = request.GET(HOST + oid[i] + "/" + aid[i] + "/", params);
			
			if (result.statusCode == 301 || result.statusCode == 302) {
				grinder.logger.warn("Warning. The response may not be correct. The response code was {}.", result.statusCode);
			} else {
				assertThat(result.statusCode, is(200));
			}
		}
	}
}
```

---

## 4. 결과

### 4-1. Redis 없을 때

- 평균 TPS: *439.1*
- 최고 TPS: *516.5*
- 평균 테스트 시간: 1,010.3ms

### 4-2. Redis 있을 때

- 평균 TPS: *1,756*
- 최고 TPS: *1,943*
- 평균 테스트 시간: 252.6ms