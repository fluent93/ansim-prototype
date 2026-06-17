# 안심 비서(Ansim) — 말로 하는 어르신 디지털 비서

고령 부모가 **앱 사용법을 배우지 않고도** 송금·전화·일정 같은 디지털 기본 생활을 **말 한마디로** 처리하게 돕는 음성 비서. 멀리 사는 자녀는 모든 활동을 공유받아 안심합니다.

> **피벗 메모**: 처음엔 사기 방어 앱(`v1-scamguard.html`)으로 시작했으나, 81세 어머니의 진짜 고통은 "사기"보다 **이체·카톡·택시 등 디지털 기본 서비스를 못 쓰는 것**이라는 통찰에 따라 방향을 바꿨습니다. 사기 방어는 이 비서의 **안전 가드 기능**으로 흡수됩니다.

- **타깃**: 디지털 소외 고령자(사용자) + 자녀(결제자)
- **수익 모델**: 구독, 월정액 (지불자=자녀)
- **첫 기능(현재 데모)**: **송금/이체** — 고통·불안이 가장 크고, 오픈뱅킹 API로 합법 실현 가능

## 실행 방법

`index.html`을 더블클릭해 브라우저에서 엽니다.

- 가운데 **🎤 마이크 버튼**을 누르고 *"둘째딸한테 오만원 보내줘"* 처럼 말하면 (크롬 등 음성 지원 브라우저), 큰 글씨 확인 화면 → 송금 완료로 이어집니다.
- 음성이 안 되는 환경에서는 **"자주 보내는 사람"** 아이콘을 눌러 동일한 흐름을 볼 수 있습니다.
- 오른쪽 **자녀 알림** 패널에 송금/확인요청이 실시간으로 쌓입니다.

### 꼭 보여줄 포인트 (안전 가드)
- **처음 보내는 계좌** → ⛔ 차단하고 "자녀 확인"으로 유도 (사기 방어 흡수)
- **30만원 이상 고액** → ⚠️ 자녀 확인 권유
- **자주 보내던 분** → ✅ 안심 송금

## 기존 데모 (참고)
- `v1-scamguard.html` — 초기 사기 탐지 콘셉트. 안전 가드 로직의 원형입니다.

## AI / API 연결 (데모 → 실서비스)

데모는 ① 음성=브라우저 Web Speech API, ② 명령 해석=규칙 기반 `parseAmount/findPerson`, ③ 이체=화면 시뮬레이션 으로 동작합니다. 실서비스 교체 지점:

### 1) 명령 해석 → Claude API
"막내아들한테 우유값 좀 보내줘" 같은 모호·구어체도 해석하고, 되물을 말까지 생성합니다.

```js
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic(); // ANTHROPIC_API_KEY

const TRANSFER_TOOL = {
  name: "prepare_transfer",
  description: "어르신의 자연어 송금 요청에서 수취인과 금액을 추출한다.",
  input_schema: {
    type: "object",
    properties: {
      recipient_alias: { type: "string", description: "사용자가 부른 이름(예: 둘째딸, 관리비)" },
      amount_krw: { type: "integer", description: "원 단위 금액. 불명확하면 0" },
      needs_clarification: { type: "string", description: "되물을 질문(한국어 존댓말). 없으면 빈 문자열" }
    },
    required: ["recipient_alias", "amount_krw", "needs_clarification"]
  }
};

export async function parseTransfer(utterance, contacts) {
  const res = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 512,
    tools: [TRANSFER_TOOL],
    tool_choice: { type: "tool", name: "prepare_transfer" },
    system: `당신은 한국 고령자의 송금 비서입니다. 등록된 수취인: ${JSON.stringify(contacts)}. ` +
            `구어체·사투리·오인식을 관용적으로 해석하되, 사람이나 금액이 불분명하면 추측하지 말고 되물으세요.`,
    messages: [{ role: "user", content: utterance }]
  });
  return res.content.find(c => c.type === "tool_use").input;
}
```

### 2) 실제 이체 → 금융결제원 오픈뱅킹 API
- 사용자 동의·인증(전자금융거래법 준수) 후 발급된 토큰으로 **출금이체/입금이체** API 호출
- **안전 가드는 서버에서 강제**: 처음 보내는 계좌·고액·이상 패턴은 송금 보류 후 자녀 승인(2차 인증)으로 전환
- 핵심: AI는 "해석"만, **돈 움직이는 결정/한도는 항상 규칙+인증 레이어**가 통제

### 3) 음성 품질 (선택)
브라우저 STT 대신 서버 STT/TTS를 쓰면 인식률·자연스러움이 올라갑니다. 핵심 흐름은 동일.

## 로드맵

1. **MVP(현재)**: 음성 송금 + 안전 가드 + 자녀 알림 — 오픈뱅킹 API 되는 것부터
2. **생활 확장**: 전화 걸기, 문자, 약/일정 알림, 날씨·정보 검색 (OS 기능 + LLM, 난이도 낮음)
3. **카톡·택시**: 공식 오픈 API 부재 → ① 제휴 ② 온디바이스 접근성(화면조작) 에이전트 ③ 자녀 co-pilot 중 택일/병행
4. **확장**: 구독 결제, 형제 공동 보호, 사기 신고 DB(경찰청·금감원) 연동

## 시장/리스크 메모
- 고령화 + 디지털 전환 가속 → 디지털 소외는 구조적·상시적 고통 (사기보다 빈도·강도 큼)
- 결제자(자녀)≠사용자(부모) → 구독 의향 높고 해지율 낮음
- **최대 리스크 = 돈·규제**: 전자금융거래법, 본인인증, 사고 책임. 그래서 "AI 전권 자동화"가 아니라 **확인·승인·한도 중심 설계**가 신뢰의 핵심
- iOS는 자동화 제약이 커 초기엔 안드로이드 우선
