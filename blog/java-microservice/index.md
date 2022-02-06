---
order: 11
layout: post
title: "자바 기반의 마이크로서비스 이해와 아키텍처 구축하기"
subtitle: "책 내용 정리 및 실무 적용을 위한 조건들"
tag: Tech Notes
type: tech-notes
blog: true
text: true
visible: true
author: JIYUN LEE
post-header: true
header-img: img/01-1.png
next-link: "../restful-api/"
prev-link: "../naver-intern-epilogue/"
---

## 1. 기본 베이스 지식

- 소프트웨어 아키텍처: 소프트웨어를 구성하는 요소들, 그리고 각 요소간의 관계를 정의해 그려놓은 것
- 4+1 View: 아키텍처의 표현을 5가지 관점에서 접근해 표현한 것
	1. Logic View: 소프트웨어를 구성하는 요소들의 관계 구조 표현
	2. Process View: 소프트웨어 간의 동적 흐름, 스레드나 프로세스 등의 동시성 처리 표현
	3. Implementation View: 논리적인 설계가 실제로 구현된 측면에서 소프트웨어의 구성과 구조 표현
	4. Deployment View: 시스템에서의 소프트웨어 배치 표현
	5. Usecase View: 시스템의 사용 사례 표현
- 소프트웨어의 기능적 측면 뿐만 아니라, 전체적인 뼈대(아키텍처)를 제대로 설계해야 프로그램을 안정적으로 연결하고 운영할 수 있다
- 아키텍처 스타일: 특정 제약 조건에서 문제 상황을 해결할 접근 방법을 제공, 그러나 정답을 주진 않음 (MSA도 기존 모놀리식 구조의 문제점 해결을 위한 접근방법에 해당하는 스타일)
- 아키텍처 패턴: 구체적 문제점에 대한 k해결 방안을 경험적 사례 기반으로 제시함

---

## 2. 모놀리식 아키텍처 vs MSA

- **모놀리식 아키텍처**: 모든 업무 로직이 하나의 큰 어플리케이션 형태로 패키징되어 서비스됨 (데이터도 한 곳에 모인 데이터를 참조)
- **마이크로서비스 아키텍처**: 작은 단위로 동작하는 서비스(작은 서비스를 식별하기 위한 이론은 Domain Driven Design 참고)가 구동되도록 시스템 및 소프트웨어의 구성과 구성 요소 간의 관계를 정의하고 설계하는 방식 (어플리케이션 기능 뿐만 아니라 데이터도 분리)
- 차이점: *하나*의 어플리케이션 vs 분할된 *다수*의 서비스와 데이터

---

## 3. SOA vs MSA

- SOA(Service Oriented Architecture): 전체 시스템을 데이터 중심이 아닌 서비스 중심으로 설계하는 스타일
- SOA의 특징
	1. 서비스 계약: 서비스는 소비자와 약속한 기능을 수행해야 하고, 소비자는 서비스 사용 계약 규칙을 준수 → 서비스 개선에 따라 여러 버전의 계약 발생
	2. 서비스 가용성: 서비스 타임아웃 기능 및 서비스 라우팅 구현을 권장 → L4/L7과 같은 하드웨어 라우팅 || circuit breaker와 같은 소프트웨어 기능적 라우팅
	3. 보안: 서비스 간 호출 시 인증 및 권한 확인 제어권을 서비스 자체에 넘기는 방식 권장
	4. 트랜잭션: Read용 DB와 Write용 DB 분리 권장 (7:3 규칙)
	5. 서비스 관리: 서비스 상태 모니터링 및 장애 대응

<div class="tw-element tw-table">
	<table class="uk-table-small uk-table style-2 uk-table-striped">
		<thead>
			<tr>
				<th> \ </th>
				<th>MSA</th>
				<th>SOA</th>
			</tr>
		</thead>
		<tbody>
			<tr>
				<td><em>사상</em></td>
				<td>서비스 지향</td>
				<td>서비스 지향</td>
			</tr>
			<tr>
				<td><em>서비스 오너십</em></td>
				<td>팀 단위 자율성 부여</td>
				<td>팀간 협업</td>
			</tr>
			<tr>
				<td><em>서비스 크기</em></td>
				<td>SOA 대비 작음</td>
				<td>MSA 대비 큼</td>
			</tr>
			<tr>
				<td><em>서비스 공유 정보</em></td>
				<td>서비스 간 독립</td>
				<td>서비스 공유</td>
			</tr>
			<tr>
				<td><em>서비스 공유 방식</em></td>
				<td>API</td>
				<td>서비스 공유를 위한 미들웨어</td>
			</tr>
			<tr>
				<td><em>서비스 통신 방식</em></td>
				<td>RESTful API 등</td>
				<td>SOAP, WSDL, UDDI, ESB 등</td>
			</tr>
		</tbody>
	</table>
</div>

---

## 4. 왜 MSA인가

- *독립적*으로 실행되면서 다른 서비스에 영향을 주지 않기 때문 → loose coupling, high cohesion을 서비스/시스템 레벨에서 구현
- 구현, 배포, 실행, 장애 발생에서 각 서비스가 독립적 → 장애 확산 방지, 빠른 장애 대응, 유연한 확장, 잦은 변경 배포에 용이
- 클라우드 네이티브 환경에 적합 → *즉시성*, *유연성*

---

## 5. 왜 클라우드 네이티브인가

- Scale-out에 유연하고, 서비스 변화 및 장애로 인한 영향을 최소화하도록 설계됨
- Docker 컨테이너 기반 서비스 운영 && CI/CD (ex. 카나리 배포, Blue-Green 배포)

---

## 6. Software as a Service

- SaaS: 특정 기간 or 기능만 필요한 만큼 구매하여 사용하는 주문형 서비스 형태의 클라우드 네이티브 소프트웨어 (MSA가 SaaS 어플리케이션 개발에 적합)
- 12 Factor
	1. 코드베이스: 하나의 어플리케이션은 하나의 코드베이스를 가지고 버전으로 관리되어야 함 (Gitlab)
	2. 종속성: 어플리케이션에 사용된 라이브러리들끼리 명시적으로 선언되지 않은 종속성이 발생하면 안 됨
	3. 환경 설정: config 정보는 코드와 완전히 분리되어 관리되어야 함
	4. 백엔드 서비스: DB, 메시지 큐와 같이 연결하거나 분리할 때 코드 수정 없이 독립적으로 수행할 수 있어야 함
	5. 빌드, 릴리즈, 실행: 코드는 빌드 → 릴리즈 → 실행 단계로 분리되어 운영됨
	6. 프로세스: 프로세스는 stateless해야 하며, 상태 정보를 보관해야 할 필요가 있다면 DB를 사용해야 함 (e.g. sticky session 지양)
	7. 포트 바인딩: 포트 바인딩된 독립 서비스들은 다른 서비스의 백엔드 서비스로도 활용될 수 있음
	8. 동시성: 특정 서비스 시스템에 부하 발생 시 scale-up보다 scale-out할 수 있는 프로세스 모델 형태 필요
	9. 폐기 가능: graceful shutdown이 보장되어야 함
	10. 개발, 테스트, 운영환경의 일관성: 각 환경이 최대한 같아야 함
	11. 로그: 어플리케이션에서 로그 처리/가공/관리에 관여해서는 안 됨 → 스트림 이벤트로 취급하여 흘려보내고, 이를 수집하는 별도 툴을 이용해 관리해야 함
	12. 일회성 프로세스: 일회성 작업은 구분하여 별도의 프로세스로 구성해야 함 (e.g. 관리자 관련 작업, 유지보수 관련 작업)

---

## 7. 마이크로서비스로의 전환

- 도메인에 따라 서비스 경계를 식별/평가해야 함 (e.g. 조직을 기준으로 서비스를 구성, 서비스 중요도 기준으로 서비스의 경계를 구분)
- 서비스 분할
	- 하향식 접근: 업무 흐름 및 중요도 기준으로 서비스 분할
	- 상향식 접근: 데이터 특성을 고려해 서비스 분할
	- 점진적 접근: 비즈니스 영향도가 적은 서비스부터 점진적으로 분할
- 조직 구성, Agile 방법론, DevOps 체계, 인프라(클라우드 네이티브 기술 환경)가 함께 고려되어야 함
- 커피 전문점 예제 ([전체 아키텍처 이미지](https://img1.daumcdn.net/thumb/R720x0.q80/?scode=mtistory2&fname=http%3A%2F%2Fcfile7.uf.tistory.com%2Fimage%2F993002505C89B48D0DE0B6))

---

## 8. 현실은

- 짧은 구글링 결과, MSA를 도입한 후기에서 아래와 같은 공통 허들 발견
	- 팀원이 전체적으로 실력이 좋아야 함
	- 좋은 커뮤니케이션이 필수적으로 전제되어야 함
	- 기존 SI나 프로젝트 성격의 개발 후 서비스 담당멤버를 운영인력으로 교체하는 방식에는 매우 적합하지 않음
	- 서비스 별로 다른 기술 스택을 가져갈 수 있는 MSA 특성으로 인해 운영 관점에서 여러 기술을 동시에 다뤄야 함 (장점으로 볼 수도 있음)
	- 서비스 간 일정 관리가 복잡해짐
	- 장애 관리가 더 어려워지는 경우 발생 (e.g. dependency 이슈 [Resiliency and high availability in microservices](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/resilient-high-availability-microservices))

---

## References
- [MSA에 대한 이해와 컨테이너 기술의 활용 방안.pdf](https://www.itfind.or.kr/WZIN/jugidong/1887/file2645276227345330267-188702.pdf)
- [Spring Camp 2018 - MSA 관련 세션 후기](https://gwonsungjun.github.io/articles/2018-04/springCamp2018)
- [MicroServices at Netflix - challenges of scale](https://www.slideshare.net/stonse/microservices-at-netflix)
