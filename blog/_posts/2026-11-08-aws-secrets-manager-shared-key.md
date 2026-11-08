---
title: "Secrets Manager로 여러 서비스가 같은 키 쓰기 — yml 환경변수, Parameter Store와 차이부터 키 교체까지"
date: 2026-11-08 10:00:00 +0900
categories: [AWS]
---
{% raw %}

외부 API 인증 키나 암호화 키를 코드 상수로 두고 있다가, 이걸 **AWS Secrets Manager**로 옮기는 이야기를 하면서 헷갈렸던 것들을 정리한다.

- yml에 `${ENV}`로 환경변수를 넣는 것과 Parameter Store, Secrets Manager는 뭐가 다른가
- Secrets Manager를 "구성했다"는 건 정확히 어디까지를 말하나, 그러면 앱은 값을 어떻게 받나
- **인프라(계정)가 서로 다른 여러 서비스**가 같은 키를 써야 하면 어떤 구조로 두나
- 그 키를 **서비스를 멈추지 않고** 바꾸려면 무엇을 준비해야 하나

> 글에 나오는 서비스·시크릿·필드 이름, 계정 ID, ARN은 모두 예시로 바꿔 적었다. 코드는 구조를 보여 주기 위한 예시 코드다.

## 1. 먼저 헷갈리는 것: "주입 방식"과 "보관 장소"는 다른 이야기다

```yaml
# application.yml
partner:
  api-key: ${PARTNER_API_KEY}
```

이렇게 쓰면 흔히 "환경변수로 뺐다"고 말한다. 그런데 이 yml이 말하는 건 **"OS 환경변수 `PARTNER_API_KEY`를 읽겠다"**뿐이다.
값은 여전히 어딘가에 있어야 하고, 누군가 그 값을 환경변수에 넣어 줘야 한다. 이 "어딘가"가 Parameter Store나 Secrets Manager다.

| 방식 | 실제 값이 있는 곳 | 문제 |
|---|---|---|
| ① 코드·yml에 값 직접 | Git 레포 | 레포를 볼 수 있으면 누구나 키를 본다. 바꾸려면 코드 수정 + 배포 |
| ② yml은 `${ENV}`만 | 서버·컨테이너 환경변수 | 코드에서는 빠졌지만 **환경변수에 넣는 값의 출처**가 따로 필요하다 |
| ③ Parameter Store / Secrets Manager | AWS의 암호화된 저장소 | ②의 출처가 되거나, 앱이 직접 읽는다 |

즉 `${ENV}`는 **어떻게 받느냐(주입)**의 문제고, Parameter Store·Secrets Manager는 **어디에 두느냐(보관)**의 문제다.
"yml에 환경변수를 넣는 게 Parameter Store인가?"에 대한 답은 **아니다**. 둘은 같이 쓰일 수 있을 뿐이다.

## 2. Parameter Store와 Secrets Manager

둘 다 "설정 값을 AWS에 두고 IAM으로 읽기 권한을 준다"는 점은 같다. Parameter Store도 `SecureString` 타입이면 KMS로 암호화된다.

| | SSM Parameter Store | Secrets Manager |
|---|---|---|
| 용도 | 설정 값 전반 (URL, 플래그, 비밀 값도 가능) | **비밀 값 전용** |
| 비용 | 표준 파라미터는 무료 | 시크릿 1개당 월 약 $0.40 + API 호출 비용 |
| 값 크기 | 표준 4KB / 고급 8KB | 64KB (JSON 통째로 넣기 좋다) |
| 자동 교체(rotation) | 없음 | Lambda로 주기 교체, RDS 등은 관리형 교체 지원 |
| 버전 | 파라미터 히스토리 | `AWSCURRENT` / `AWSPREVIOUS` 같은 **버전 단계(staging label)** |
| 다른 계정과 공유 | 고급 티어 + AWS RAM 공유 | **리소스 기반 정책**으로 직접 허용 |

고르는 기준은 이렇다.

- 값을 **우리가 주기적으로 바꿀 수 있고 바꿔야 한다** (DB 비밀번호 등) → Secrets Manager의 자동 교체가 의미 있다.
- **외부 파트너가 발급한 키**처럼 우리 마음대로 못 바꾸는 값 → 자동 교체는 못 쓴다. 비용만 보면 Parameter Store로도 충분하다.
- **여러 계정이 같은 값을 읽어야 한다**, JSON 구조로 여러 값을 묶어 두고 싶다 → Secrets Manager가 편하다.

## 3. "Secrets Manager를 구성했다"는 건 어디까지인가

누군가(보통 인프라 담당) Secrets Manager를 구성했다는 건 다음 두 가지를 했다는 뜻이다.

1. **보관**: 시크릿을 만들고 값(JSON)을 넣었다.
2. **권한**: 누가 읽을 수 있는지 정책을 열었다.

여기까지는 "공통 공간에 값을 넣어 두고 문을 열어 둔 것"이다. **앱이 그 값을 받아 쓰는 길은 따로 정해야 한다.** 길은 크게 두 가지다.

```
                 ┌─ (A) ECS/EKS가 꺼내서 환경변수로 주입 ─→ 앱은 ${ENV}로 읽는다
Secrets Manager ─┤
                 └─ (B) 앱이 SDK/Spring으로 직접 조회 ───→ 환경변수가 필요 없다
```

### (A) 배포 플랫폼이 환경변수로 넣어 주기

yml은 1번 예시처럼 `${PARTNER_API_KEY}`만 두고, 값은 ECS 태스크 정의에서 넣는다.

```json
"containerDefinitions": [{
  "name": "order-api",
  "secrets": [{
    "name": "PARTNER_API_KEY",
    "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:111111111111:secret:shared/partner-auth-AbCdEf:apiKey::"
  }]
}]
```

- `valueFrom` 끝의 `:apiKey::`는 JSON 시크릿 안에서 `apiKey` 필드만 꺼내라는 뜻이다.
- 컨테이너가 뜰 때 ECS가 값을 꺼내 환경변수로 넣는다. 그래서 읽기 권한은 앱의 태스크 역할이 아니라 **태스크 실행 역할(execution role)**에 있어야 한다.
- EKS라면 External Secrets Operator가 Secrets Manager 값을 K8s Secret으로 동기화하고, 그걸 환경변수로 넣는 방식을 많이 쓴다.
- 앱 코드는 Secrets Manager의 존재를 모른다. 대신 값은 **컨테이너가 뜰 때 한 번** 들어가므로, 키가 바뀌면 재시작해야 반영된다.

### (B) 앱이 직접 읽기

Spring Cloud AWS를 쓰면 yml에 **시크릿 주소만** 적으면 된다.

```yaml
spring:
  config:
    import: aws-secretsmanager:arn:aws:secretsmanager:ap-northeast-2:111111111111:secret:shared/partner-auth-AbCdEf
```

시크릿이 `{"apiKey": "..."}` 같은 JSON이면, 앱이 뜰 때 각 필드가 프로퍼티로 등록되어 `@Value("${apiKey}")`로 받을 수 있다.
ARN은 비밀이 아니다. Git에 있어도 **IAM 권한이 없으면 읽을 수 없다.**

값 안에서 고를 것이 있거나 재시작 없이 바뀐 값을 반영하고 싶으면 SDK로 직접 조회한다. 이건 5장에서 다룬다.

| | (A) 환경변수 주입 | (B) 앱이 직접 조회 |
|---|---|---|
| 설정 주체 | 인프라 (태스크 정의, 오퍼레이터) | 개발자 (yml에 ARN, 코드) |
| 권한이 필요한 역할 | ECS 실행 역할 / 오퍼레이터 | 앱이 실행되는 태스크 역할(IRSA) |
| 키가 바뀌면 | 재시작해야 반영 | 캐시가 만료되면 반영 |
| 어울리는 값 | 값 하나, 거의 안 바뀜 | JSON 안에서 골라 써야 하는 값, 무중단 교체 |

## 4. 계정이 다른 여러 서비스가 같은 키를 읽는 구조

서비스 A·B·C가 각자 다른 AWS 계정(그리고 ECS, EKS, EC2처럼 다른 실행 환경)에 있는데 같은 키를 써야 한다고 하자.
키를 계정마다 복사해 두면 바꿀 때 세 군데를 똑같이 고쳐야 하고, 하나라도 빠지면 장애가 난다.
그래서 **한 계정(공통 계정)에 시크릿을 하나만 두고, 나머지 계정이 읽어 가게** 한다.

```
           ┌─────────────── 공통 계정 (시크릿 소유) ───────────────┐
           │  Secrets Manager: shared/partner-auth                 │
           │  + 고객 관리형 KMS 키 (CMK)                            │
           │  + 리소스 정책: 아래 세 역할만 GetSecretValue 허용       │
           └───────▲──────────────────▲──────────────────▲─────────┘
                   │ 교차 계정          │                  │
         서비스 A 계정 (ECS)   서비스 B 계정 (EKS)   서비스 C 계정 (EC2)
         task role            IRSA role            instance role
```

교차 계정으로 읽으려면 **세 군데가 모두 열려 있어야** 한다.

**① 시크릿의 리소스 정책** (공통 계정)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": [
      "arn:aws:iam::222222222222:role/service-a-task",
      "arn:aws:iam::333333333333:role/service-b-irsa",
      "arn:aws:iam::444444444444:role/service-c-ec2"
    ]},
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "*"
  }]
}
```

**② KMS 키 정책** (공통 계정): 같은 역할들에 `kms:Decrypt`를 허용한다.

**③ 각 서비스 계정의 IAM 정책**: 자기 역할에 공통 계정 시크릿 ARN에 대한 `GetSecretValue`, 그 CMK에 대한 `kms:Decrypt`를 준다.

여기서 자주 막히는 두 가지가 있다.

- **기본 키(`aws/secretsmanager`)로 암호화한 시크릿은 다른 계정에서 읽을 수 없다.**
  AWS 관리형 키는 키 정책을 고칠 수 없어서 다른 계정에 `Decrypt`를 열어 줄 방법이 없다. 그래서 공유할 시크릿은 처음부터 **CMK로 암호화**해야 한다.
- **다른 계정에서는 시크릿 이름으로 찾을 수 없다.** 이름은 자기 계정 안에서만 통하므로 **전체 ARN**으로 조회해야 한다.
  ARN 끝에는 AWS가 붙인 6자리 접미사(`-AbCdEf`)가 있으니 그것까지 포함한 값을 받아 둔다.

실행 환경이 ECS든 EKS든 EC2든 결국 "IAM 역할로 같은 ARN을 읽는다"는 점은 같다. 실행 환경마다 다른 건 그 역할을 어떻게 붙이느냐(태스크 역할, IRSA, 인스턴스 프로파일) 정도다.

## 5. 키를 바꿔야 할 때: 구 키와 신 키가 함께 있어야 한다

여러 서비스가 같은 **암호화 키**를 쓴다면, 한쪽이 암호화한 데이터를 다른 쪽이 복호화한다.
키를 바꾸는 순간 아직 이전 키로 암호화된 데이터가 남아 있으므로, **한동안은 두 키가 모두 필요하다.**

Secrets Manager의 `AWSCURRENT`/`AWSPREVIOUS`는 **시크릿 값 전체의 버전**이다. "이 데이터는 이전 값으로 풀고, 저 데이터는 새 값으로 풀어라"를 대신 판단해 주지 않는다.
그래서 흔히 **시크릿 JSON 안에 버전별 키를 같이 넣고, 어떤 키를 쓸지는 앱이 고르게** 한다.

```json
{
  "activeVersion": "v2",
  "keys": {
    "v1": "<기존 키 (Base64)>",
    "v2": "<신규 키 (Base64)>"
  }
}
```

규칙은 두 줄이다.

- **암호화**는 항상 `activeVersion`의 키로 한다.
- **복호화**는 그 데이터를 암호화한 버전의 키로 한다. 그러려면 **데이터에 버전이 같이 붙어 있어야** 한다.

### 암호문에 버전을 붙인다

```java
public String encrypt(String plain) {
    KeySet keySet = keyRing.current();
    String version = keySet.activeVersion();

    byte[] iv = new byte[12];
    secureRandom.nextBytes(iv);

    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    cipher.init(Cipher.ENCRYPT_MODE, aesKey(keySet.key(version)), new GCMParameterSpec(128, iv));
    byte[] encrypted = cipher.doFinal(plain.getBytes(StandardCharsets.UTF_8));

    // "v2:<iv>:<암호문>" — 어떤 키로 암호화했는지 데이터가 들고 다닌다
    return version + ":" + b64(iv) + ":" + b64(encrypted);
}

public String decrypt(String token) {
    String[] parts = token.split(":", 3);
    String version = parts[0];

    KeySet keySet = keyRing.current();
    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    cipher.init(Cipher.DECRYPT_MODE, aesKey(keySet.key(version)),   // 없는 버전이면 예외
            new GCMParameterSpec(128, unb64(parts[1])));
    return new String(cipher.doFinal(unb64(parts[2])), StandardCharsets.UTF_8);
}
```

버전을 암호문 안에 넣기 어려우면 API 요청에 버전 필드를 같이 넘겨도 된다. 핵심은 **복호화하는 쪽이 키를 추측하지 않게 하는 것**이다.

### "새 키로 풀어 보고 실패하면 옛 키로"의 함정

버전 정보를 붙일 수 없으면 "신규 키로 먼저 복호화하고, 실패하면 기존 키로 재시도"를 생각하게 된다. 여기엔 함정이 있다.

- **AES-CBC, ECB처럼 무결성 검증이 없는 방식**은 틀린 키로 복호화해도 예외 없이 깨진 값을 돌려줄 수 있다.
  CBC는 패딩 검사에 걸리면 예외가 나지만, 우연히 패딩 모양이 맞으면 그냥 통과한다. "예외가 안 났다 = 맞는 키"가 아니다.
- 그래서 이 방식을 쓴다면 **복호화 결과가 기대한 형식인지**(자릿수, 체크섬, 문자 범위 등) 반드시 같이 검증해야 한다.
- 근본적인 해결은 위 예시처럼 **AES-GCM 같은 인증 암호화(AEAD)**를 쓰는 것이다. 틀린 키면 인증 태그 검증에서 반드시 `AEADBadTagException`이 나므로, 재시도 판단이 확실해진다.

### 키를 읽어 오는 부분: 캐시하고 주기적으로 다시 읽는다

`GetSecretValue`를 요청마다 부르면 비용과 지연이 생기고 API 호출 한도에도 걸린다. 몇 분짜리 캐시를 둔다.

```java
@Component
public class KeyRing {

    private static final Duration TTL = Duration.ofMinutes(5);

    private final SecretsManagerClient client;
    private final ObjectMapper objectMapper;
    private final String secretArn;      // yml에는 이 ARN만 둔다

    private volatile Loaded loaded;

    public KeySet current() {
        Loaded snapshot = loaded;
        if (snapshot == null || snapshot.isExpired()) {
            synchronized (this) {
                if (loaded == null || loaded.isExpired()) {
                    loaded = new Loaded(fetch(), Instant.now().plus(TTL));
                }
                snapshot = loaded;
            }
        }
        return snapshot.keySet();
    }

    private KeySet fetch() {
        String json = client.getSecretValue(r -> r.secretId(secretArn)).secretString();
        return objectMapper.readValue(json, KeySet.class);
    }

    private record Loaded(KeySet keySet, Instant expiresAt) {
        boolean isExpired() { return Instant.now().isAfter(expiresAt); }
    }
}
```

직접 만들기 싫으면 AWS가 제공하는 캐시 라이브러리(`aws-secretsmanager-caching-java`)를 써도 된다.

### 교체 순서: 한 번에 바꾸면 안 된다

캐시가 있으므로 서비스마다 새 값을 읽는 시점이 다르다. 그래서 순서가 중요하다.

| 단계 | 시크릿 상태 | 이유 |
|---|---|---|
| 1. 새 키 **추가만** | `active: v1`, keys: v1, v2 | 모든 서비스가 v2로 **복호화할 수 있게** 먼저 퍼뜨린다 |
| 2. 캐시 TTL 이상 대기 | | 모든 서비스가 새 값을 읽을 때까지 |
| 3. active 전환 | `active: v2`, keys: v1, v2 | 이제부터 v2로 암호화한다 |
| 4. 기존 데이터 정리 후 v1 삭제 | `active: v2`, keys: v2 | v1로 암호화된 데이터가 남아 있지 않을 때만 |

1번과 3번을 한 번에 하면, 아직 v1만 아는 서비스가 다른 서비스가 v2로 암호화한 데이터를 받아 **풀지 못한다.**

### 키의 용도를 나누면 교체 범위가 줄어든다

서비스 사이에 데이터를 넘길 때 쓰는 **전달용 암호화 키**와, 오래 저장할 데이터를 만드는 키(예: 토큰을 발급하는 별도 키 관리 시스템)를 나눠 두면 좋다.
전달용 키는 넘기는 순간에만 쓰이므로, 바꿔도 **이미 저장된 데이터를 다시 암호화할 필요가 없다.** 키 하나가 여러 역할을 맡을수록 교체할 때 건드려야 할 범위가 커진다.

## 6. 그런데 "하나의 키를 여러 서비스가 공유"가 최선일까

위 구조는 "여러 서비스가 같은 키를 써야만 한다"는 전제에서의 정리다. 외부 API 인증 키처럼 선택의 여지가 있다면 **호출하는 쪽마다 키를 따로 발급받는 것**이 더 낫다.

| | 공유 키 1개 | 호출자별 키 |
|---|---|---|
| 유출이 의심될 때 | 어느 서비스에서 샜는지 알기 어렵다 | 해당 키만 폐기하면 된다 |
| 키를 바꿀 때 | 모든 서비스가 같이 움직여야 한다 | 서비스별로 따로 바꾼다 |
| 감사·추적 | 받는 쪽 로그로는 호출자를 구분할 수 없다 | 키 단위로 호출자를 식별한다 |
| 권한 | 모두 같은 권한 | 서비스마다 허용 범위를 다르게 줄 수 있다 |

공유해야 한다면 최소한 CloudTrail로 **누가 언제 `GetSecretValue`를 호출했는지** 남겨 두고, 리소스 정책의 허용 대상을 꼭 필요한 역할로만 좁힌다.

## 흔한 실수

- **yml에 `${ENV}`만 써 놓고 "Secrets Manager 적용했다"고 생각한다.** 그 환경변수를 누가 어디서 채우는지까지 정해야 끝이다.
- **기본 KMS 키로 만든 시크릿을 다른 계정과 공유하려 한다.** 리소스 정책을 다 열어도 복호화 단계에서 `AccessDenied`가 난다.
- **ECS 주입 방식인데 태스크 역할에만 권한을 준다.** 컨테이너 시작 시 값을 꺼내는 건 실행 역할이다. 반대로 앱이 SDK로 읽는다면 태스크 역할에 권한이 있어야 한다.
- **키를 바꾸고 바로 이전 키를 지운다.** 이전 키로 암호화된 데이터가 남아 있으면 그 데이터는 영영 못 푼다.
- **비밀 값을 로그에 남긴다.** 시크릿 JSON을 통째로 `log.debug` 하거나, 설정 값을 덤프하는 actuator 엔드포인트를 열어 두면 저장소를 바꾼 의미가 없다.

## 한 줄 정리

> yml의 `${ENV}`는 "어떻게 받느냐", Parameter Store·Secrets Manager는 "어디에 두느냐"의 이야기다.
> 계정이 다른 여러 서비스가 같은 키를 쓰려면 공통 계정에 CMK로 암호화한 시크릿 하나를 두고, 리소스 정책·KMS 키 정책·각 계정 IAM을 모두 열어 전체 ARN으로 읽게 한다.
> 키를 무중단으로 바꾸려면 JSON 안에 버전별 키를 두고, 암호문에 버전을 붙이고, "추가 → 전파 대기 → 전환 → 정리" 순서로 바꾼다.

## 참고

- [AWS Secrets Manager — 다른 계정의 사용자가 시크릿에 액세스하도록 허용](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_examples_cross.html)
- [Amazon ECS — Secrets Manager를 사용하여 민감한 데이터 전달](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data-secrets.html)
- [Spring Cloud AWS — Secrets Manager Integration](https://docs.awspring.io/spring-cloud-aws/docs/3.0.0/reference/html/index.html#secrets-manager-integration)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
{% endraw %}
