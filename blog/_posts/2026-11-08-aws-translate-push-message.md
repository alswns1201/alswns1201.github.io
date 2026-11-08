---
title: "Amazon Translate로 알림 문구 번역하기 — 같은 문장은 항상 같게 번역될까"
date: 2026-11-08 11:00:00 +0900
categories: [AWS]
---
{% raw %}

앱 푸시 같은 알림을 사용자 언어로 보내려고 할 때, 문구마다 번역가를 붙이기는 어렵다. 그래서 **Amazon Translate**로 기계 번역을 붙이는 방법을 정리한다.

정리하다 보니 가장 먼저 궁금했던 건 이것이다.

> **같은 문장을 넣으면 매번 정확히 같은 단어로 번역될까?**

결론부터 말하면 **"대체로 같지만, 보장되지 않는다"**. 그래서 설계는 "보낼 때마다 번역"이 아니라 **"한 번 번역해서 저장해 두고 그걸 보낸다"**로 가는 게 맞다. 왜 그런지, 그리고 그걸 전제로 어떻게 구성하는지 순서대로 적는다.

> 글에 나오는 서비스·테이블·필드 이름과 알림 문구는 모두 예시다. 코드는 구조를 보여 주기 위한 예시 코드다.

## Amazon Translate란

AWS의 **신경망 기계 번역(NMT)** 서비스다. 원문과 원문 언어, 목표 언어를 주면 번역문을 돌려준다.

| 방식 | API | 용도 |
|---|---|---|
| 실시간 | `TranslateText` | 요청 1건에 텍스트 최대 10,000바이트(UTF-8). 바로 결과를 받는다 |
| 실시간 문서 | `TranslateDocument` | HTML·docx 등 문서 하나를 바로 번역 |
| 비동기 배치 | `StartTextTranslationJob` | S3에 올린 대량 문서를 잡으로 번역. 한 잡에 목표 언어 최대 10개 |

비용은 **번역한 글자 수** 기준이다(표준 번역 기준 100만 자당 약 $15). 알림 문구는 짧아서 글자 수 자체는 적지만, **사용자마다 매번 번역**하면 같은 문장에 계속 비용을 내게 된다. 이것도 "한 번 번역해서 저장"으로 가는 이유 중 하나다.

## 같은 문장은 항상 같게 번역될까

### 대체로는 같다

Translate는 생성형 AI 챗봇처럼 "온도(temperature)"를 두고 무작위로 고르는 방식이 아니다. 같은 모델에 같은 입력과 같은 옵션을 주면 **대부분 같은 결과**가 나온다. 짧은 시간 안에 같은 문장을 여러 번 보내 보면 보통 같은 번역이 돌아온다.

### 하지만 보장하지 않는다

AWS 문서 어디에도 **"같은 입력이면 같은 출력"을 보장한다는 말은 없다.** 결과가 달라질 수 있는 지점은 이렇다.

1. **모델이 바뀐다.** AWS는 번역 품질을 계속 개선한다. 그런데 API에는 **모델 버전을 고정하는 옵션이 없다.** 어제 "할인 쿠폰"으로 나오던 단어가 모델 업데이트 후 "할인권"으로 나올 수 있다.
2. **옵션이 다르면 결과가 다르다.** 용어집(custom terminology), 격식(formality), 간결하게(brevity), 병렬 데이터(parallel data)를 쓰는지에 따라 같은 문장도 다르게 나온다. 서버마다 설정이 조금씩 다르면 같은 문장이 서로 다르게 번역된다.
3. **같은 단어도 문장이 다르면 다르게 번역된다.** 문장 전체 맥락으로 번역하기 때문에, "포인트"가 어떤 문장에서는 "point", 다른 문장에서는 "credit"으로 나올 수 있다. 문장이 같아도 앞뒤에 한 글자만 바뀌면 단어 선택이 달라질 수 있다.
4. **용어집도 항상 먹히는 건 아니다.** 용어집을 넣었는데 어떤 입력에서는 적용되고 어떤 입력에서는 안 되는 사례가 AWS 커뮤니티에 올라와 있다. 문장 구조에 따라 용어가 활용형으로 바뀌면 매칭이 안 되기도 한다.

그래서 **"같은 문장은 같게 나오겠지"를 전제로 시스템을 짜면 안 된다.** 특히 알림은 운영자가 보낸 문구와 사용자가 받은 문구가 같아야 하고, 문제가 생기면 "그때 무엇이 나갔는지"를 설명할 수 있어야 한다. 매번 번역하면 그걸 증명할 수 없다.

### 그래서: 번역은 한 번, 결과는 저장

```
[관리자] 알림 문구 등록/수정 (원문: 한국어)
     │
     ▼
[서버] 지원 언어마다 Translate 호출 → 번역문 저장 (원문 해시와 함께)
     │                                   └ 관리자가 확인·수정 가능
     ▼
[배치] 발송 시점: 사용자 언어로 저장된 번역문을 꺼내서 보냄  ← 여기서는 Translate를 부르지 않는다
```

- **발송 시점에는 Translate를 부르지 않는다.** 저장된 번역문을 쓰므로 같은 알림은 누구에게나 같은 문장으로 나간다.
- 번역이 이상하면 **관리자가 저장된 번역문을 고친다.** 기계 번역을 초안으로 쓰고, 사람이 최종 확인하는 구조다.
- 원문이 바뀌면 원문 해시가 바뀌므로 그때만 다시 번역한다.
- 발송 시점에 Translate가 느리거나 실패하거나 한도에 걸려도 **발송에는 영향이 없다.**

## 사용자 언어 고르기: OS 언어 → Translate 언어 코드

앱에서 받은 OS 언어 값은 형식이 제각각이다. iOS는 `zh-Hans-KR`, Android는 `zh_CN`이나 `zh-TW` 같은 식이다. Translate가 받는 코드와 그대로 맞지 않으니 **매핑 함수**를 하나 둔다.

```java
public final class LanguageResolver {

    private static final String DEFAULT = "en";
    private static final Set<String> SUPPORTED = Set.of("ko", "en", "ja", "zh", "zh-TW");

    public static String resolve(String osLocale) {
        if (osLocale == null || osLocale.isBlank()) {
            return DEFAULT;
        }
        Locale locale = Locale.forLanguageTag(osLocale.replace('_', '-'));
        String lang = locale.getLanguage();          // "zh", "ko", "en" ...

        if ("zh".equals(lang)) {
            // 번체: zh-Hant, zh-TW, zh-HK → zh-TW / 그 외 → 간체 zh
            String script = locale.getScript();
            String country = locale.getCountry();
            boolean traditional = "Hant".equals(script) || "TW".equals(country) || "HK".equals(country);
            return traditional ? "zh-TW" : "zh";
        }
        return SUPPORTED.contains(lang) ? lang : DEFAULT;
    }
}
```

- 중국어는 **간체(`zh`)와 번체(`zh-TW`)를 반드시 나눈다.** 대만·홍콩 사용자에게 간체로 보내면 번역을 안 한 것만 못하다.
- 포르투갈어(`pt` / `pt-PT`), 프랑스어(`fr` / `fr-CA`), 스페인어(`es` / `es-MX`)처럼 지역 변형이 있는 언어도 있다. **지원할 언어 목록을 먼저 정하고** 그 안에서만 매핑한다.
- 매핑이 안 되면 기본 언어로 보낸다. 지원하지 않는 언어 사용자에게 알림이 아예 안 가는 것보다 낫다.

## 번역 호출 (AWS SDK for Java v2)

```java
@Component
@RequiredArgsConstructor
public class MessageTranslator {

    private final TranslateClient translateClient;

    public String translate(String text, String targetLanguage) {
        TranslateTextResponse response = translateClient.translateText(r -> r
                .text(text)
                .sourceLanguageCode("ko")                 // 원문 언어를 명시한다 ("auto"는 짧은 문장에서 틀리기 쉽다)
                .targetLanguageCode(targetLanguage)
                .terminologyNames("notice-terms")         // 용어집 (요청당 1개)
                .settings(s -> s.formality(Formality.FORMAL)));   // 격식 — 지원하는 목표 언어에서만 의미가 있다
        return response.translatedText();
    }
}
```

- 원문 언어를 `auto`로 두면 자동 감지를 한다. 알림 문구처럼 짧은 문장은 감지가 틀릴 수 있고, 감지가 다르면 결과도 달라진다. **원문 언어를 아는 경우에는 명시한다.**
- `Formality`는 일부 목표 언어에서만 지원한다. 지원하지 않는 언어에 넣으면 오류가 날 수 있으니 언어별로 켜고 끈다.

### 저장할 때 같이 남길 것

```
notice_translation
  notice_id       알림 ID
  language        ko / en / ja / zh / zh-TW
  source_hash     번역할 때의 원문 해시 (원문이 바뀌었는지 판단)
  text            번역문 (관리자가 고칠 수 있음)
  translated_by   MACHINE / ADMIN
  updated_at
```

`translated_by`가 `ADMIN`이면 관리자가 직접 고친 번역이다. 원문이 바뀌지 않았으면 다시 번역해도 덮어쓰지 않는다.

## 번역되면 안 되는 부분 지키기

알림 문구에는 번역되면 안 되는 부분이 섞여 있다.

### 1) 치환 변수: `{userName}`님, `{couponName}`

`{userName}`님, 쿠폰이 도착했어요 같은 문장을 그대로 넣으면, 기계 번역이 중괄호 안을 번역하거나 중괄호를 바꿔 버릴 수 있다.
치환 후에 번역하면 사람 이름까지 번역되고, 사용자마다 번역해야 하니 앞의 "한 번만 번역" 원칙도 깨진다.

Translate는 **HTML 입력에서 `translate="no"`가 붙은 요소는 번역하지 않는다.** 변수를 그 태그로 감싸서 보내고, 결과에서 태그를 걷어 낸다.

```java
private static final Pattern VAR = Pattern.compile("\\{(\\w+)}");

String protect(String text) {
    // HTML로 보내므로 &, <, > 는 먼저 이스케이프
    String escaped = HtmlUtils.htmlEscape(text);
    return VAR.matcher(escaped).replaceAll("<span translate=\"no\">{$1}</span>");
}

String restore(String translated) {
    String unwrapped = translated.replaceAll("<span translate=\"no\">(\\{\\w+})</span>", "$1");
    return HtmlUtils.htmlUnescape(unwrapped);
}
```

번역문이 저장되기 전에 **원문과 번역문의 변수 목록이 같은지 검사**한다. 하나라도 빠지거나 바뀌었으면 저장하지 않고 관리자에게 확인을 요청한다.

### 2) 고유 용어: 서비스 이름, 메뉴 이름

서비스 이름이나 앱 메뉴 이름처럼 **항상 같은 단어로 나가야 하는 용어**는 용어집(custom terminology)으로 고정한다. CSV로 만들어 등록한다.

```csv
ko,en,ja
마이포인트,My Points,マイポイント
적립금,Rewards,ポイント残高
```

다만 앞에서 말했듯 용어집도 문장에 따라 적용이 안 될 수 있다. 용어집은 **번역 초안의 품질을 올리는 도구**로 보고, 최종 확인은 저장 단계의 검증과 관리자 검토에 맡긴다.

## 흔한 실수

- **발송할 때마다 번역한다.** 비용이 사용자 수만큼 늘고, 같은 알림이 사람마다 다른 문장으로 나갈 수 있으며, Translate 장애가 곧 발송 장애가 된다.
- **번역 후 길이를 확인하지 않는다.** 한국어보다 영어·독일어가 훨씬 길어지는 경우가 많다. 푸시 제목·본문은 기기에서 잘리므로 저장할 때 길이를 검사한다(간결하게 옵션 `brevity`도 있다).
- **중국어를 하나로 취급한다.** 간체와 번체는 따로 지원해야 한다.
- **개인정보를 번역 요청에 넣는다.** Translate는 서비스 개선을 위해 입력을 저장할 수 있다(AWS Organizations의 AI 서비스 옵트아웃 정책으로 끌 수 있다). 사용자 이름처럼 개인을 식별할 수 있는 값은 치환 변수로 남겨 두고 번역 요청에는 보내지 않는다.
- **"같은 입력이면 같은 출력"을 테스트로 고정한다.** 모델이 바뀌면 테스트가 깨진다. 테스트는 Translate를 목(mock)으로 바꾸고, 실제 번역 품질은 사람이 확인한다.

## 한 줄 정리

> Amazon Translate는 같은 입력에 대체로 같은 번역을 주지만, 모델 업데이트·옵션·문맥 때문에 **항상 같다고 보장하지 않는다.**
> 그래서 알림 문구는 **등록할 때 언어별로 한 번 번역해서 저장**하고, 발송할 때는 사용자 OS 언어를 지원 언어 코드로 매핑해 저장된 번역문을 쓴다.
> 치환 변수는 `translate="no"`로 보호하고, 고유 용어는 용어집으로 고정하되, 최종 품질은 저장 전 검증과 관리자 검토로 지킨다.

## 참고

- [Amazon Translate — Guidelines and quotas](https://docs.aws.amazon.com/translate/latest/dg/what-is-limits.html)
- [Amazon Translate — Customizing your translations](https://docs.aws.amazon.com/translate/latest/dg/customizing-translations.html)
- [Amazon Translate — Using do-not-translate](https://docs.aws.amazon.com/translate/latest/dg/customizing-translations-tags.html)
- [Amazon Translate FAQ](https://aws.amazon.com/translate/faqs/)
- [AWS re:Post — inconsistent results with amazon translate and terminology](https://repost.aws/questions/QUpn9n9SYhSZCMWg9EVgrSuw/inconsistent-results-with-amazon-translate-and-terminology)
{% endraw %}
