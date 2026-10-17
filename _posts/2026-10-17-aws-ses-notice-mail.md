---
title: "AWS SES로 안내 메일 보내기 — AWS SDK v2 SesClient 적용"
date: 2026-10-17
categories: [AWS]
---
{% raw %}

서비스에서 사용자에게 안내 메일을 보내야 해서 **Amazon SES(Simple Email Service)**를 붙였다.
SMTP 서버를 직접 운영하지 않고, Spring 서버에서 **AWS SDK for Java v2의 `SesClient`**로 SES API를 호출하는 방식이다.
디자이너에게 받은 HTML을 **SES 템플릿**으로 등록하고, SES 쪽 준비부터 의존성, `SesClient` 설정, 템플릿 발송, 이벤트 리스너 처리까지 순서대로 정리해 둔다.

> 글에 나오는 도메인, 메일 주소, 템플릿·클래스 이름과 메일 문구는 모두 예시로 바꿔 적었다.

## SES란

AWS가 제공하는 메일 발송 서비스다. 직접 메일 서버를 두면 IP 평판 관리, 스팸 필터, 반송(bounce) 처리까지 신경 써야 하는데,
SES는 이걸 맡아 주고 **보낸 만큼만 비용**을 낸다.

애플리케이션에서 SES로 메일을 보내는 방법은 두 가지다.

| 방식 | 필요한 것 | 특징 |
|---|---|---|
| SMTP 인터페이스 | SMTP 자격 증명(사용자명/비밀번호) | `spring-boot-starter-mail`(JavaMailSender)을 그대로 쓸 수 있다 |
| **SES API (AWS SDK)** | IAM 권한 | SDK로 API를 직접 호출. 별도 SMTP 비밀번호가 필요 없다 ← 이번 글 |

서버가 이미 AWS에서 돌고 있어서 IAM 역할로 권한을 주면 되니, **SMTP 비밀번호를 따로 관리하지 않아도 되는 SDK 방식**을 골랐다.

## 1. SES 쪽 준비

코드보다 먼저 콘솔에서 해야 할 일이 있다.

1. **리전 선택** — SES는 리전별로 설정이 따로다. 코드에서 쓸 리전과 같은 리전에서 설정해야 한다.
2. **자격 증명(Identity) 생성** — 보내는 주소를 인증한다.
   - **도메인** 인증: `example.com` 전체를 인증. DNS에 SES가 알려 주는 **DKIM CNAME 레코드**를 등록한다.
   - **이메일 주소** 인증: 주소 하나만 인증. 확인 메일의 링크를 누르면 끝.
   - 실서비스라면 도메인 인증을 하고, SPF·DMARC까지 맞춰 두어야 스팸함으로 덜 빠진다.
3. **샌드박스 해제 요청** — 처음 계정은 **샌드박스** 상태라서 *인증된 주소로만* 보낼 수 있고 발송량도 제한된다.
   실제 사용자에게 보내려면 콘솔에서 **프로덕션 액세스 요청**을 해야 한다(용도, 수신 동의 여부, 반송/수신거부 처리 방법 등을 적는다).
4. **IAM 권한** — 메일을 보내는 서버(EC2/ECS 역할 등)에 `ses:SendTemplatedEmail` 권한을 준다.

```json
{
  "Effect": "Allow",
  "Action": ["ses:SendEmail", "ses:SendTemplatedEmail"],
  "Resource": "*"
}
```

> 샌드박스 상태에서 테스트하면 "받는 사람 주소도 인증해야 한다"는 에러가 난다.
> 개발 중에는 받을 주소도 이메일 자격 증명으로 등록해 두고 테스트하면 된다.

## 2. 의존성 — AWS SDK v2 BOM + ses 모듈

```kotlin
// build.gradle.kts
implementation(platform("software.amazon.awssdk:bom:2.25.70"))  // AWS SDK v2 버전 일괄 관리
implementation("software.amazon.awssdk:ses")                     // SES 모듈 (v1 API)
```

- **BOM(`platform(...)`)** 은 AWS SDK v2 모듈들의 버전을 한 곳에서 맞춰 준다.
  나중에 `sqs`, `s3` 같은 모듈을 추가해도 버전을 따로 적지 않아도 되고, 모듈끼리 버전이 어긋날 일이 없다.
- `ses` 모듈 주석의 **"v1 API"는 SDK 버전이 아니라 SES API 버전**이다. 헷갈리기 쉬운 부분이라 따로 적어 둔다.

| 구분 | 의미 |
|---|---|
| AWS SDK **v1** / **v2** | 자바 SDK 자체의 버전. 패키지가 `com.amazonaws` / `software.amazon.awssdk`로 다르다. v1은 지원 종료 단계라 새로 쓴다면 v2. |
| SES API **v1** / **v2** | SES 서비스 API의 버전. SDK v2 안에 `ses` 모듈(`SesClient`)과 `sesv2` 모듈(`SesV2Client`)이 따로 있다. |

즉 지금 구성은 **"SDK는 v2, 호출하는 SES API는 v1(`SesClient`)"** 이다.
템플릿 등록·템플릿 발송(`sendTemplatedEmail`)까지 v1 API로 충분했다. 수신자 목록(contact list)이나 수신 거부 관리 같은 기능까지 쓴다면 `sesv2`를 보는 게 낫다.

## 3. SesClient 빈 등록

`SesClient`는 스레드 안전하고 내부에 HTTP 커넥션 풀을 들고 있어서, **요청마다 만들지 말고 빈으로 하나만 등록**해서 재사용한다.

```java
@Configuration
public class SesConfig {

	@Bean
	public SesClient sesClient(@Value("${aws.ses.region}") String region) {
		return SesClient.builder()
				.region(Region.of(region))
				.credentialsProvider(DefaultCredentialsProvider.create())
				.build();
	}
}
```

```yaml
# application.yml
aws:
  ses:
    region: ap-northeast-2
    from: "서비스 안내 <no-reply@example.com>"
```

- `DefaultCredentialsProvider`는 환경변수 → 시스템 프로퍼티 → `~/.aws/credentials` → 컨테이너/EC2 역할 순으로 자격 증명을 찾는다.
  **운영에서는 서버의 IAM 역할**이, 로컬에서는 `~/.aws/credentials`의 프로필이 잡히니 코드에 키를 적을 필요가 없다.
- 리전은 SES 자격 증명을 만든 리전과 같아야 한다. 다르면 "인증되지 않은 주소"라는 에러가 난다.

## 4. 메일 템플릿 만들기 — 디자이너 HTML을 SES 템플릿으로

안내 메일은 디자이너·기획자에게 **HTML 시안**으로 전달받았다.
이 HTML을 코드 안에 문자열로 넣지 않고, **SES 템플릿**으로 등록해 두고 쓰기로 했다.

- 메일 디자인(HTML)과 서버 코드가 분리된다. 문구나 디자인이 바뀌어도 **템플릿만 다시 올리면 되고 배포가 필요 없다.**
- 서버는 "어떤 템플릿에, 어떤 값을 넣을지"만 넘긴다.

### 1) 바뀌는 부분을 변수로 바꾸기

전달받은 HTML에서 사용자마다 달라지는 부분을 `{{변수}}` 자리로 바꾼다. SES 템플릿은 Handlebars 문법을 쓴다.

```html
<!-- 디자이너에게 받은 HTML (일부) -->
<p><strong>홍길동</strong>님, 신청하신 내역이 처리되었습니다.</p>
<p>처리 일시: 2026-10-17 15:00</p>

<!-- 변수로 바꾼 뒤 -->
<p><strong>{{name}}</strong>님, 신청하신 내역이 처리되었습니다.</p>
<p>처리 일시: {{processedAt}}</p>
```

### 2) JSON으로 만들어 SES에 등록

SES 템플릿은 **이름, 제목, HTML 본문, 텍스트 본문**을 묶은 JSON이다. 변경한 HTML을 `HtmlPart`에 넣는다.

```json
{
  "Template": {
    "TemplateName": "notice-complete",
    "SubjectPart": "[서비스] {{name}}님, 처리가 완료되었습니다",
    "HtmlPart": "<html><body><p><strong>{{name}}</strong>님, 신청하신 내역이 처리되었습니다.</p><p>처리 일시: {{processedAt}}</p></body></html>",
    "TextPart": "{{name}}님, 신청하신 내역이 처리되었습니다. 처리 일시: {{processedAt}}"
  }
}
```

```bash
aws ses create-template --cli-input-json file://notice-complete.json   # 처음 등록
aws ses update-template --cli-input-json file://notice-complete.json   # 수정할 때
```

- HTML을 JSON 문자열 한 줄로 넣어야 해서 **큰따옴표(`"`)는 이스케이프**해야 한다. 손으로 하면 실수하기 쉬우니 스크립트로 변환하는 게 편하다.
- **`TextPart`도 같이 넣는다.** HTML을 못 보여 주는 메일 클라이언트용이고, 스팸 점수에도 영향을 준다.
- 메일 HTML은 웹 페이지와 다르다. 외부 CSS·`<style>`이 잘 안 먹어서 **인라인 스타일**로 작성된 시안을 그대로 쓰는 게 안전하다.
- 템플릿도 **리전별**이다. `SesClient`에 설정한 리전에 등록해야 한다.

## 5. 메일 발송 클래스 — sendTemplatedEmail

템플릿을 쓰면 본문을 직접 만들지 않고 **템플릿 이름 + 변수 값(JSON)**만 보낸다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class SesMailSender {

	private final SesClient sesClient;
	private final ObjectMapper objectMapper;

	@Value("${aws.ses.from}")
	private String from;

	public void send(String to, String templateName, Map<String, String> variables) {
		SendTemplatedEmailRequest request = SendTemplatedEmailRequest.builder()
				.source(from)
				.destination(d -> d.toAddresses(to))
				.template(templateName)
				.templateData(toJson(variables))       // {"name":"홍길동","processedAt":"..."}
				.build();

		SendTemplatedEmailResponse response = sesClient.sendTemplatedEmail(request);
		log.info("SES 발송 완료 to={}, template={}, messageId={}", to, templateName, response.messageId());
	}

	private String toJson(Map<String, String> variables) {
		try {
			return objectMapper.writeValueAsString(variables);
		} catch (JsonProcessingException e) {
			throw new IllegalArgumentException("템플릿 변수 변환 실패", e);
		}
	}
}
```

- SDK v2는 빌더 안에서 **람다로 하위 객체를 바로 채울 수 있어서**(`destination(d -> ...)`) 코드가 짧아진다.
- `templateData`는 **JSON 문자열**이다. 직접 문자열을 이어 붙이면 이름에 따옴표가 들어갈 때 깨지니 `ObjectMapper`로 만든다.
- **템플릿에 있는 변수를 빠뜨리면** 호출 자체는 성공해도 SES가 렌더링 단계에서 발송을 실패 처리하고 메일이 안 나간다. 변수 이름은 템플릿과 정확히 맞춰야 한다.
- 응답의 `messageId`를 로그로 남겨 두면 나중에 "메일이 안 왔다"는 문의가 들어왔을 때 SES 쪽 기록과 맞춰 볼 수 있다.

## 6. 비즈니스 로직과 떼어 놓기 — ApplicationEventPublisher

안내 메일은 "어떤 처리가 끝났다"를 알리는 부가 기능이다. 서비스 코드에서 `SesMailSender`를 직접 부르면

- 메일 발송이 실패하면 **본 로직의 트랜잭션까지 롤백**될 수 있고,
- 반대로 트랜잭션이 롤백됐는데 **메일은 이미 나가는** 일도 생긴다.
- SES 응답을 기다리는 시간만큼 API 응답도 늦어진다.

그래서 서비스에서는 **이벤트만 발행**하고, 메일은 **커밋이 끝난 뒤** 리스너에서 보낸다.

```java
public record NoticeMailEvent(String email, String name, LocalDateTime processedAt) {}
```

```java
@Service
@RequiredArgsConstructor
public class SomeService {

	private final ApplicationEventPublisher eventPublisher;

	@Transactional
	public void complete(Long id) {
		// ... 본 로직 처리 ...
		eventPublisher.publishEvent(new NoticeMailEvent(user.getEmail(), user.getName(), LocalDateTime.now()));
	}
}
```

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class NoticeMailListener {

	private static final DateTimeFormatter FORMAT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");

	private final SesMailSender sesMailSender;

	@Async
	@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
	public void handle(NoticeMailEvent event) {
		try {
			sesMailSender.send(event.email(), "notice-complete", Map.of(
					"name", event.name(),
					"processedAt", event.processedAt().format(FORMAT)));
		} catch (Exception e) {
			log.warn("안내 메일 발송 실패 to={}: {}", event.email(), e.getMessage());
		}
	}
}
```

- **`@TransactionalEventListener(AFTER_COMMIT)`** — 트랜잭션이 커밋된 뒤에만 실행된다. 롤백되면 메일도 안 나간다.
  (그냥 `@EventListener`는 `publishEvent()`를 호출한 그 자리에서 바로 실행돼서, 커밋 전에 메일이 나갈 수 있다.)
- **`@Async`** — 별도 스레드에서 보내서 API 응답이 SES를 기다리지 않는다. `@EnableAsync` 설정이 필요하다.
- 리스너 안에서 예외를 잡아 로그만 남긴다. 이미 커밋이 끝난 뒤라 여기서 던져도 본 로직에는 영향이 없지만, 실패 기록은 남겨야 한다.
- 이벤트에는 **메일 내용이 아니라 "무슨 일이 있었는지"**만 담고, 템플릿 이름과 변수 매핑은 리스너가 정한다.
  서비스는 메일의 존재를 모르고, 나중에 알림 채널(푸시, 알림톡)이 늘어나도 리스너만 추가하면 된다.

## 주의할 점

- **템플릿을 바꿀 땐 개발 리전·계정에서 먼저 테스트 발송**해 본다. 변수 이름이 어긋나면 에러 없이 메일만 안 나가서 늦게 알아챈다.
- **샌드박스 해제 전에는 실사용자에게 못 보낸다.** 오픈 일정이 있다면 프로덕션 액세스 요청을 미리 넣어 둔다(승인까지 시간이 걸릴 수 있다).
- **발송 속도 제한**(초당 발송 수)이 계정마다 있다. 대량 발송이면 한 번에 몰아 보내지 말고 큐에 넣어 나눠 보낸다.
- **반송(bounce)·스팸 신고(complaint) 비율**이 높으면 SES가 계정 발송을 막을 수 있다. 존재하지 않는 주소로 계속 보내지 않도록, SNS 알림으로 반송을 받아 처리하는 것까지 고려한다.
- `SesClient`는 **빈 하나로 재사용**한다. 매번 `build()`하면 커넥션 풀이 계속 새로 만들어진다.

## 한 줄 정리

> SES 콘솔에서 도메인 인증·샌드박스 해제를 하고, AWS SDK v2(BOM + `ses` 모듈)의 `SesClient`를 빈으로 등록한다.
> 디자이너 HTML은 변수(`{{name}}`)를 넣어 JSON 템플릿으로 SES에 등록하고, 서버는 `sendTemplatedEmail`로 템플릿 이름과 변수 값만 보낸다.
> 발송은 `@TransactionalEventListener(AFTER_COMMIT)` + `@Async`로 본 로직에서 떼어 내서, 커밋된 경우에만 비동기로 나가게 한다.

## 참고

- [AWS SES 소개 및 설정 — 농심 클라우드 기술 블로그](https://tech.cloud.nongshim.co.kr/blog/aws/2237/)
- [Spring ApplicationEventPublisher란](https://velog.io/@dlawotn3/Spring-ApplicationEventPublisher-%EC%9D%B4%EB%9E%80)
{% endraw %}
