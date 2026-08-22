---
title: "Skill Prompt, Tool Use, MCP — LLM을 확장하는 세 가지 방식은 뭐가 다른가"
date: 2026-05-19
categories: [LLM]
---

LLM을 실제 서비스에 붙이다 보면 세 가지 개념이 계속 등장한다. Skill Prompt, Tool
Use, MCP. 이름도 쓰임도 비슷해 보이지만, "누가 무엇을 실행하는가"라는 축으로
보면 완전히 다른 층위에 있다.

## 1. Skill Prompt — 행동 규칙을 텍스트로 주입한다

`.md` 파일에 행동 규칙이나 도메인 지식을 적어두고, 그걸 System/User Prompt에
그대로 삽입하는 방식이다.

```
.md 파일 로드 → System/User Prompt에 삽입 → LLM이 읽고 규칙에 따라 응답
```

```java
@Value("classpath:skills/math-parser.md")
private Resource mathSkillResource;

// .md 파일을 읽어서 프롬프트에 삽입
this.skillPrompt = StreamUtils.copyToString(
    mathSkillResource.getInputStream(), StandardCharsets.UTF_8
);

MessageCreateParams.builder()
    .addUserMessage(skillPrompt + "\n\n" + userInput)
    .build();
```

LLM이 외부의 무언가를 호출하는 게 아니다 — 프롬프트에 담긴 지침을 읽고 **스스로
판단해서 따르는** 것뿐이다. 그래서 한계도 명확하다: LLM이 지침을 지키려고
노력하지만 보장은 없다. 문제가 복잡해질수록 형식이 흔들리거나 필드를 빠뜨릴 수
있다. 결과물이 확률적이라는 뜻이다. 분석 방법이나 답변 스타일, 도메인 규칙처럼
"어떻게 생각할지"를 지정할 때 쓴다.

## 2. Tool Use — LLM이 함수 호출을 "요청"한다

Tool Use(Function Calling)는 LLM이 외부 함수를 직접 실행하는 게 아니라, 실행을
**요청**하는 메커니즘이다. 개발자가 함수 이름·설명·입력 스키마를 JSON으로
정의해두면, LLM은 대화 중 필요하다고 판단할 때 그 함수를 이 인자로 호출해달라는
신호만 보낸다. 실제 실행은 개발자 코드가 맡는다.

```
도구 정의 (JSON 스키마)
    → LLM이 "이 도구 써야겠다" 판단
    → tool_use 블록 응답
    → 개발자 코드가 실제 API 호출
    → 결과를 LLM에 다시 전달
    → 최종 응답 생성
```

```java
// 1. 도구 정의
Tool parseTool = Tool.builder()
    .name("parse_math_problems")
    .description("이미지에서 수학 문제를 추출")
    .inputSchema(/* JSON 스키마 */)
    .build();

// 2. LLM에 도구와 함께 요청
MessageCreateParams.builder()
    .tools(List.of(ToolUnion.ofTool(parseTool)))
    .toolToolChoice("parse_math_problems") // 반드시 이 도구 사용 강제
    .build();

// 3. LLM 응답에서 tool_use 블록 꺼내서 개발자가 처리
response.content().stream()
    .flatMap(block -> block.toolUse().stream())
    .findFirst()
    .map(toolUse -> {
        // 여기서 실제 로직 실행
    });
```

Skill Prompt와 결정적으로 다른 지점은 **강제 주체**다. Skill Prompt는 LLM이
확률적으로 형식을 지키려 노력하는 반면, Tool Use는 스키마를 API 레벨에서
강제한다.

| | Skill Prompt로 형식 지정 | Tool Use 스키마 강제 |
|---|---|---|
| 강제 주체 | LLM (확률적) | API 레벨 (결정적) |
| 앞뒤 텍스트 | 붙을 수 있음 | 절대 안 붙음 |
| 필드 누락 | 가능 | required면 불가 |
| 파싱 코드 | try/catch 필수 | 바로 역직렬화 가능 |

구조화된 JSON 출력을 반드시 보장해야 하거나, 외부 API(날씨, 계산기, DB 등)를
LLM 판단에 따라 호출해야 하고, 그 호출 로직을 개발자가 직접 제어하고 싶을 때
쓴다.

## 3. MCP — 호출 로직 자체를 서버가 대신 구현해둔다

MCP(Model Context Protocol)는 Anthropic이 만든 표준 프로토콜로, LLM과 외부
서비스를 연결하는 방식을 규격화한 것이다. Tool Use와 MCP의 차이는 "중간 코드를
누가 짜는가"에 있다.

```
Tool Use:
LLM → tool_use 신호 → 개발자 코드 → 외부 API

MCP:
LLM → MCP 서버 → 외부 API
      (이미 구현된 서버)
```

Tool Use는 외부 API를 호출하는 중간 코드를 개발자가 직접 작성해야 한다. MCP는
그 중간 코드가 MCP 서버 안에 이미 구현되어 있어서, 개발자는 서버에 연결만 하면
된다.

```
MCP 서버 URL 등록 (한 번만)
    → LLM이 대화 중 필요하다고 판단
    → MCP 서버에 직접 요청
    → MCP 서버가 외부 API 호출 및 결과 반환
    → LLM이 결과로 최종 응답 생성
```

```
// 연결만 해두면 끝
mcp_servers: [
    { type: "url", url: "https://gmailmcp.googleapis.com/mcp/v1", name: "gmail-mcp" },
    { type: "url", url: "https://drivemcp.googleapis.com/mcp/v1", name: "drive-mcp" }
]
// LLM이 필요하면 알아서 Gmail, Drive 호출
```

Gmail, Google Drive, Slack처럼 이미 MCP 서버가 존재하는 서비스를 연동할 때,
호출 타이밍과 로직을 LLM에게 완전히 위임하고 싶을 때, 여러 서비스를 하나의
표준 방식으로 연결하고 싶을 때 쓴다.

## 세 가지 비교

| | Skill Prompt | Tool Use | MCP |
|---|---|---|---|
| 역할 | 행동 규칙 주입 | 함수 호출 메커니즘 | 외부 서비스 연결 표준 |
| 실행 주체 | LLM (추론) | 개발자 코드 | MCP 서버 |
| 호출 코드 작성 | 불필요 | 개발자가 직접 작성 | MCP 서버가 담당 |
| 결과 보장 | 확률적 | 결정적 (스키마 강제 시) | 결정적 |
| 재사용성 | 프롬프트 복사 | 앱마다 재구현 | 서버 하나로 공유 |

## 대립이 아니라 계층이다

셋은 서로 대체하는 관계가 아니라, 계층적으로 함께 쓰인다.

```
Skill Prompt  →  "수학 문제를 이렇게 분석해"       (행동 지침)
Tool Use      →  분석 결과를 JSON으로 반드시 반환   (출력 보장)
MCP           →  필요하면 외부 DB에 저장            (서비스 연결)
```

각각의 역할이 명확히 분리되어 있어서, "어떻게 생각할지"는 Skill Prompt가,
"결과를 어떻게 확정할지"는 Tool Use가, "외부 세계와 어떻게 연결할지"는 MCP가
맡는다. 셋 중 하나만 골라야 하는 문제가 아니라, 어떤 층위의 문제를 풀고
있는지를 먼저 구분하는 게 먼저다.
