---
title: "AI 에이전트 3-Way Fan-out 개발기 — hocg 스킬 설계와 운영"
excerpt: "같은 질문을 3개 AI 에이전트에 동시에 던지고 결과를 합성하는 hocg 스킬의 설계 과정, 구현 상세, Anti-hang 안전장치, 실제 운영 사례와 한계를 정리합니다. 이후 4-way 구성으로 확장한 과정도 덧붙입니다."
categories:
  - Development
tags:
  - Multi-Agent
  - Orchestration
  - Claude Code
  - Fan-out
  - Shell Script
  - Home Lab
toc: true
toc_sticky: true
sidebar:
  nav: "docs"
---

## 개요

[멀티 에이전트 오케스트레이션](/development/multi-agent-orchestration/) 글에서 hocg를 간략하게 언급했다. 이 글에서는 설계 동기부터 구현 상세, 안전장치, 실제 운영에서 드러난 한계까지 풀어본다.

hocg는 **H**ermes + **o**mo + Open**C**law + Claude 합성(**g** = glue)의 약어다. 3개의 외부 워커에 동일한 질문을 동시에 보내고, Claude Code(3호기)가 결과를 합성하는 fan-out 패턴이다.

핵심 목표는 두 가지다.
1. **다관점 검증**: 단일 모델의 hallucination이나 누락을 교차 검증으로 잡아낸다.
2. **쿼터 절약**: Claude Code(Opus 4.7)는 합성만 담당해 Max 쿼터 소모를 최소화한다.

---

## 설계 배경: /ccg에서 /hocg로

### 이전 버전: /ccg

최초에는 Claude Code + Codex + Gemini 조합의 `/ccg` 스킬이 있었다. 하지만 Codex 환경을 실제로 거의 쓰지 않았고, Gemini(Antigravity)는 IDE 안에서만 동작해 CLI 호출이 어려웠다. 결국 `/ccg`는 사실상 사장된 기능이 됐다.

### 재설계 동기

에이전트 fleet이 6개로 늘어나며 환경이 바뀌었다:

| 변화 | 내용 |
|------|------|
| Hermes(5호기) 투입 | Qwen3.5-397B 로컬 서빙, 무한 쿼터, CLI 호출 가능 |
| OpenCode/omo 구성 | Oh-My-OpenAgent로 Kimi K2.6 + GPT-5.5 라우팅, CLI 호출 가능 |
| OpenClaw(2호기) SSH 접근 | Mac에서 DeepSeek V4 Pro 실행, SSH 경유 CLI 호출 가능 |

세 워커 모두 **CLI에서 비대화형으로 호출 가능하다**는 공통점이 있었다. 이 조건이 fan-out 패턴의 전제조건이다.

---

## 아키텍처

### 전체 흐름

```mermaid
sequenceDiagram
    participant U as 사용자
    participant CC as Claude Code (3호기)<br/>합성자
    participant HM as Hermes (5호기)<br/>Qwen3.5-397B
    participant OMO as OpenCode (omo)<br/>Kimi K2.6
    participant OC as OpenClaw (2호기)<br/>DeepSeek V4 Pro

    U->>CC: /hocg "아키텍처 리뷰 요청"
    
    Note over CC: 0. Pre-check (hocg-doctor.sh)
    Note over CC: 1. Decompose: 워커별 프롬프트 작성
    
    par 2. Parallel Call
        CC->>HM: hocg-call-hermes.sh
        CC->>OMO: hocg-call-omo.sh
        CC->>OC: hocg-call-openclaw.sh (SSH)
    end
    
    HM-->>CC: hermes-{ts}.md
    OMO-->>CC: omo-{ts}.md
    OC-->>CC: openclaw-{ts}.md
    
    Note over CC: 3. Collect: 아티팩트 3개 수집
    Note over CC: 4. Synthesize: 합성 답변 생성
    
    CC->>U: 통합 답변 + 아티팩트 경로
```

### 역할 분리 원칙

Claude Code(3호기)는 **오케스트레이터이자 합성자**다. 실제 추론 작업은 워커가 수행하며, 3호기는 프롬프트 분배와 결과 합성만 담당한다. 이렇게 분리한 이유는 Opus 4.7의 Max 플랜 쿼터를 절약하기 위해서다.

워커별 강점에 따라 프롬프트를 분기할 수 있지만, 보통은 동일한 프롬프트를 보낸다. 의사결정이나 아키텍처 리뷰에서 다양한 관점을 확보하는 것이 주 목적이기 때문이다.

| 워커 | 모델 | 강점 | 비용 |
|------|------|------|------|
| Hermes (5호기) | Qwen3.5-397B int4, vLLM | 깊은 추론, 긴 컨텍스트 분석, 아키텍처 리스크 평가 | 0원 (자체호스팅) |
| OpenCode (omo) | Kimi K2.6 / GPT-5.5 (Ultraworker) | 코딩, 도구 호출, 단일 파일 변경 | 일부 유료 |
| OpenClaw (2호기) | DeepSeek V4 Pro (Ollama Cloud) | 일반 추론, UX 관점, 문서 표현 | 0원 (Ollama Cloud) |

---

## 구현 상세

### Wrapper 스크립트 구조

각 워커는 독립된 bash wrapper 스크립트로 호출한다. 인터페이스는 다음과 같이 통일했다:

```bash
hocg-call-{worker}.sh "<PROMPT>" "<OUTPUT_PATH>"
```

- **입력**: 프롬프트 문자열 + 결과를 저장할 파일 경로
- **출력**: 워커의 응답이 지정된 파일에 기록됨
- **종료 코드**: 0=성공, 2=바이너리 미존재, 3=네트워크 미도달(OpenClaw 전용)

이 단순한 인터페이스 덕분에 워커를 교체하거나 추가할 때 wrapper만 새로 작성하면 된다.

### 워커별 호출 방식

**Hermes (로컬 바이너리)**

가장 단순한 방법이다. hermes CLI 바이너리에 `-z` (non-interactive) 플래그를 설정하고 stdout을 파일로 리다이렉트한다.

```bash
exec timeout --kill-after="$KILL_AFTER" "$TIMEOUT" \
  "$HERMES" -z "$PROMPT" </dev/null >"$OUT" 2>&1
```

**OpenCode/omo (로컬 바이너리)**

opencode CLI의 `run` 서브커맨드로 비대화형 실행을 한다. omo 플러그인이 자동 로드되어 Sisyphus Ultraworker 라우팅이 적용된다.

```bash
exec timeout --kill-after="$KILL_AFTER" "$TIMEOUT" \
  "$OPENCODE" run "$PROMPT" </dev/null >"$OUT" 2>&1
```

**OpenClaw (SSH 원격 호출)**

가장 복잡한 단계다. Mac 호스트에 SSH로 접속해 OpenClaw agent를 실행해야 한다. 프롬프트에 특수문자가 포함될 수 있어 `printf %q`로 이스케이프 처리한다.

```bash
ESCAPED=$(printf %q "$PROMPT")
exec timeout --kill-after="$KILL_AFTER" "$TIMEOUT" ssh \
  -o ConnectTimeout="$SSH_CONNECT_TIMEOUT" \
  -o ServerAliveInterval="$SSH_ALIVE_INTERVAL" \
  -o ServerAliveCountMax="$SSH_ALIVE_COUNT_MAX" \
  -o BatchMode=yes \
  -i "$SSH_KEY" \
  "$SSH_HOST" \
  "$NODE_BIN $OPENCLAW_JS agent -m $ESCAPED --json --agent $OPENCLAW_AGENT" \
  </dev/null >"$OUT" 2>&1
```

### 병렬 실행과 Watchdog

세 wrapper를 백그라운드(`&`)로 동시 실행하고, `wait`로 모두 종료될 때까지 기다린다. 여기에 전체 데드라인 watchdog 프로세스를 추가해 어떤 상황에서도 시간 상한을 보장한다.

```bash
TS=$(date -u +%Y%m%dT%H%M%SZ)
ART=.omc/artifacts/ask
mkdir -p "$ART"
OVERALL_DEADLINE="${HOCG_OVERALL_DEADLINE:-645}"

bash ~/scripts/hocg-call-hermes.sh   "$HERMES_PROMPT"   "$ART/hermes-$TS.md"   &
PA=$!
bash ~/scripts/hocg-call-omo.sh      "$OMO_PROMPT"      "$ART/omo-$TS.md"      &
PB=$!
bash ~/scripts/hocg-call-openclaw.sh "$OPENCLAW_PROMPT" "$ART/openclaw-$TS.md" &
PC=$!

# 전체 deadline watchdog
( sleep "$OVERALL_DEADLINE" && kill -KILL $PA $PB $PC 2>/dev/null ) &
WATCHDOG=$!

wait $PA $PB $PC 2>/dev/null
kill $WATCHDOG 2>/dev/null
```

타임아웃 체계는 3단계로 구성된다.

| 계층 | 기본값 | 역할 |
|------|--------|------|
| wrapper `timeout` | 600s | 개별 워커의 실행 시간 상한 |
| `--kill-after` | 15s | SIGTERM 무시 시 SIGKILL까지 대기 |
| OVERALL_DEADLINE watchdog | 645s | 전체 fan-out의 wall-clock 상한 |

---

## Anti-hang 안전장치

외부 프로세스를 병렬로 호출하는 구조에서 가장 위험한 것은 **무한 대기**다. 어떤 하나의 워커가 hang 상태에 빠지면 전체 세션이 멈춘다. 이를 방지하려고 5가지 안전장치를 적용했다.

### 1. timeout + kill-after

모든 wrapper는 `timeout --kill-after=$HOCG_KILL_AFTER $HOCG_TIMEOUT` 으로 감싸져 있다.

- SIGTERM을 보내고 `$HOCG_KILL_AFTER`(15초) 동안 종료를 기다린다
- 15초 후에도 살아있으면 SIGKILL로 강제 종료한다

### 2. stdin 차단

</dev/null`로 stdin을 차단한다. 이 설정이 없으면 CLI 도구가 대화형 프롬프트를 띄워 무한 대기에 빠질 수 있다. 실제로 초기 개발 당시 hermes가 `--yes or --no?` 프롬프트를 띄워 hang이 발생했다.

### 3. 바이너리 존재성 사전 검증

wrapper 실행 직후 바이너리 존재 여부를 확인한다. 바이너리가 없으면 exit code 2로 즉시 종료한다. 600초 타임아웃을 기다리지 않고 빠르게 실패 처리한다.

### 4. SSH 도달성 사전 검증 (OpenClaw 전용)

OpenClaw는 Mac에 SSH로 접속해야 하므로, 호출 전 `nc -z -w 2` 로 포트 22 도달 여부를 확인한다. Mac이 절전 모드이거나 네트워크가 끊겼다면, 2초 만에 실패하고 exit code 3을 반환한다.

### 5. SSH KeepAlive (OpenClaw 전용)

SSH 연결이 성립된 후에도 stall이 발생할 수 있다. `ServerAliveInterval=15`와 `ServerAliveCountMax=4`를 설정하면, 15초마다 alive 패킷을 보내고 4회 연속 응답이 없을 때 SSH 연결을 끊는다.

이 5가지를 조합하면, **어떤 상황에서도 wrapper는 `HOCG_TIMEOUT + HOCG_KILL_AFTER` 시간 내에 반드시 종료된다**고 보장한다.

---

## Pre-check: hocg-doctor

fan-out 실행 전 `hocg-doctor.sh`를 실행해 워커 가용성을 미리 점검한다.

```text
$ ~/scripts/hocg-doctor.sh
[A. Hermes] OK (hermes 바이너리 존재)
[B. omo]    OK (opencode 바이너리 존재)
[C. OpenClaw] OK (Mac 호스트 SSH 포트 도달)

summary: 3/3 workers available
```

exit code에 따른 행동:

| Exit Code | 의미 | 행동 |
|-----------|------|------|
| 0 | 3/3 가용 | 정상 3-way fan-out |
| 1 | 1~2개 가용 | 가용 워커만 호출 (degraded mode) |
| 2 | 0개 가용 | Claude Code가 직접 답변으로 폴백 |

doctor 스크립트는 Hermes와 omo의 바이너리 존재 여부를, OpenClaw는 SSH 포트 도달성을 각각 점검한다. 전체 점검 시간은 3초 이내다.

---

## 합성 프로토콜

세 워커의 응답을 수집하면, Claude Code가 다음 6개 항목으로 합성을 수행한다:

### 합성 구조

1. **결론 한 줄**: 세 응답의 공통 결론을 한 문장으로 요약
2. **동의 영역**: 3개 워커 모두 일치한 부분
3. **불일치 영역**: 서로 다른 부분. "Hermes는 X를 권장, omo는 Y를 제안, OpenClaw는 Z를 주장" 형태로 각 워커를 명시
4. **추천안**: Claude Code의 종합 판단. 어느 워커의 의견에 가중치를 두는지와 그 이유
5. **후속 행동**: 사용자가 다음에 취할 단계
6. **아티팩트 경로**: 세 원본 응답의 절대 경로. 사용자가 직접 검증할 수 있도록 cite

불일치 영역이 핵심이다. 단일 모델에 질문하면 한 가지 답만 받지만, 3-way에서는 **모델 간 견해가 다른 부분**이 자연스럽게 드러나 사용자가 더 나은 판단을 내린다.

### Graceful Degradation

모든 워커가 항상 정상 응답하지는 않는다. 실패 상황별 처리:

- **1개 워커 실패**: 가용한 2개 워커의 응답으로 합성 수행. 누락된 워커를 명시
- **2개 워커 실패**: 남은 1개 워커의 응답을 그대로 사용자에게 전달. 합성은 수행하지 않음
- **모두 실패**: "3 워커 모두 응답 실패" 보고. 사용자에게 Claude Code 직접 답변 여부를 질문

실제 운영에서 가장 흔한 실패 원인은 OpenClaw의 Mac 절전 모드다. doctor 사전 점검으로 대부분 걸러지지만, doctor와 실제 호출 사이에 Mac이 절전에 들어가는 경우도 있다. 이때는 SSH 연결 타임아웃으로 감지된다.

---

## 사용 사례

### 적합한 사례

- **아키텍처 결정**: "이 서비스를 모노리스로 갈지 마이크로서비스로 갈지" 같은 설계 판단에서 3개 모델의 서로 다른 관점이 가치 있었다
- **코드 리뷰**: 동일한 PR diff를 세 모델에 보내면, 각각 다른 관점에서 문제를 지적한다. 한 모델이 놓친 보안 이슈를 다른 모델이 잡아내는 경우가 있었다
- **기술 조사**: 특정 기술의 장단점 분석에서 모델별 학습 데이터 차이로 인한 다양한 시각 확보

### 부적합한 사례

- **코드 변경 작업**: 세 모델이 각각 다른 코드를 생성하면 합치는 것이 불가능하다. 코드 변경은 단일 에이전트에게 맡기는 것이 맞다
- **단순 정보 조회**: "K8s에서 pod를 삭제하려면?" 같은 1-shot 질문에 3-way를 쓰는 것은 낭비다
- **긴급 작업**: 가장 느린 워커에 응답 시간이 바운드되므로, 빠른 응답이 필요한 작업에는 적합하지 않다

---

## 한계와 교훈

### 레이턴시

3개 워커를 병렬로 호출하지만, 응답 시간은 **가장 느린 워커**에 따라 결정된다. 현재 구성에서는 Hermes(397B 모델)의 초기 프리필 시간이 병목이다. 짧은 질문이라도 30초~1분 정도 소요되며, 긴 컨텍스트를 보내면 수 분이 걸린다.

### 합성 품질의 의존성

최종 결과물의 품질은 Claude Code의 합성 능력에 크게 의존한다. 워커 3개의 응답 포맷이 제각각이면(한 워커는 마크다운 테이블, 다른 워커는 산문체, 또 다른 워커는 코드 블록) 통합 난도가 올라간다. 프롬프트에 "마크다운으로 답변하라"는 지시를 넣어 포맷을 어느 정도 통일하는 것이 실용적이었다.

### 쿼터 관리

워커 3개 중 2개(Hermes, OpenClaw)는 무료지만, omo는 Kimi K2.6이나 GPT-5.5 쿼터를 소모한다. 빈번하게 호출하면 비용이 발생하므로, 실제로는 중요한 의사결정에만 선택적으로 사용한다.

### 코드 변경의 한계

가장 근본적인 한계다. hocg는 **의견을 모으는 도구**이지 **작업을 수행하는 도구**가 아니다. "이 함수를 리팩토링해줘"라는 요청을 3개 모델에 보내면 세 가지 다른 코드가 나오며, 이를 합치는 것은 불가능하다. 코드 변경은 단일 에이전트에게 맡기고, hocg는 "어떻게 리팩토링할지" 방향 설정에만 사용하는 것이 맞다.

### 환경 의존성

OpenClaw 워커가 Mac에 SSH로 접속하는 구조라, Mac이 절전 모드이거나 네트워크가 변경되면 해당 워커를 사용할 수 없다. 워커 가용성이 물리적 환경에 의존하는 점은 취약점이다.

---

## 설정과 커스터마이징

모든 동작 파라미터는 환경변수로 override 가능하다.

| 환경변수 | 기본값 | 설명 |
|---------|--------|------|
| `HOCG_TIMEOUT` | 600 | 개별 워커 타임아웃 (초) |
| `HOCG_KILL_AFTER` | 15 | SIGTERM 후 SIGKILL까지 대기 (초) |
| `HOCG_OVERALL_DEADLINE` | 645 | 전체 fan-out wall-clock 상한 (초) |
| `HOCG_SSH_KEY` | (사용자 SSH 키) | OpenClaw SSH 키 경로 |
| `HOCG_OPENCLAW_HOST` | (Mac 호스트) | OpenClaw SSH 호스트 |
| `HOCG_PRECHECK_TIMEOUT` | 2 | SSH 도달성 사전 점검 타임아웃 (초) |
| `HOCG_DOCTOR_TIMEOUT` | 3 | doctor 점검 타임아웃 (초) |

타임아웃을 줄이면 빠르게 실패하지만, Hermes처럼 초기 로딩이 긴 모델은 정상 응답도 잘릴 수 있다. 현재 600초(10분)는 397B 모델의 긴 프리필까지 고려한 보수적인 값이다.

---

## 마무리

hocg는 "AI 에이전트에게 물어보는 것도 한 번 더 물어보면 낫다"는 단순한 아이디어에서 출발했다. 구현 자체는 bash wrapper 4개와 합성 프로토콜이 전부지만, 안전장치를 빠뜨리면 CLI 세션이 멈추는 실제 장애로 이어진다는 점을 학습하는 데 시간이 걸렸다.

현재 가장 효과적인 사용 패턴은 **아키텍처 의사결정 직전**에 hocg를 한 번 실행하는 것이다. 세 모델의 견해가 일치하면 확신을 갖고 진행하며, 불일치하면 더 깊이 검토해야 할 지점이 명확해진다.

코드 변경에는 적합하지 않고 레이턴시 오버헤드가 있으며, 물리적 환경에 의존하는 취약점도 있다. 하지만 중요한 설계 결정에서 "한 모델의 의견만 듣고 결정했다"는 불안을 덜어주는 것만으로도 충분히 가치가 있다.

---

## 4-Way로의 진화 (2026-07 업데이트)

초판(2026-05-15)의 hocg는 Hermes·omo·OpenClaw 3-way 구성이었다. 이후 두 달 동안 백엔드 지형이 바뀌면서 구성 자체를 재편했다. 앞선 3-way 서사는 당시의 기록으로 그대로 두고, 여기서는 무엇이 어떻게 달라졌는지만 덧붙인다.

### 문제: 워커 간 품질 편차 확대

트리거는 DGX Spark 서빙 모델 전환이었다. 클러스터의 대형 모델 서빙이 deepseek-v4-flash(컨텍스트 524K)로 넘어가면서(2026-06-01), Hermes 워커가 의존하던 대형 모델 백엔드가 로컬 Ollama의 gemma4:31b로 강등됐다.

결과적으로 Hermes의 추론 품질이 눈에 띄게 떨어졌다. fan-out은 "세 워커가 비슷한 체급에서 서로 다른 관점을 낼 때" 교차 검증이 성립한다. 그런데 한 워커만 체급이 내려가자, 합성 단계에서 Hermes의 응답이 다른 두 워커 대비 얕거나 빗나가는 경우가 늘었다. 다관점 검증이라는 본래 목적이 오히려 편차 노이즈로 오염되기 시작한 것이다.

### 의사결정: Hermes 제외 (2026-06-16)

품질 편차를 몇 차례 관측한 뒤, Hermes 워커를 fan-out에서 제외했다(2026-06-16). "워커 수가 많을수록 낫다"는 직관과는 반대되는 판단이지만, 체급이 맞지 않는 워커를 억지로 끼워 넣는 것은 합성 품질을 떨어뜨릴 뿐이었다. 스킬 이름 hocg의 첫 글자 H(ermes)는 이 시점부터 이름에만 남은 흔적(legacy)이 됐다.

### 보강: 이종 벤더 워커 추가

3-way에서 2-way로 줄이는 대신, 관점 다양성을 "같은 로컬 모델 여러 개"가 아니라 "서로 다른 벤더 계열"에서 확보하는 방향으로 재설계했다.

- **antigravity (2026-06-28 추가)**: Google Gemini 계열. Anthropic·Kimi·DeepSeek와 학습 데이터·정렬 방향이 다른 벤더를 넣어 관점의 결을 벌렸다.
- **codex (2026-07-02 추가)**: GPT-5.x 계열. 정밀 리뷰·코드 관점 보강용. 다만 codex는 별도 리뷰 게이트(`/codex:review`)와 쿼터를 공유하므로, fan-out에서는 대량 호출을 피하고 정밀 보강 슬롯으로만 제한했다.

### 결과: 이종 4-Way 교차 검증

현재 hocg는 omo·OpenClaw·antigravity·codex 4개 워커에 동일 질문을 fan-out하고 Claude(3호기)가 합성하는 구조다.

| 워커 | 모델 계열 | 관점 |
|------|-----------|------|
| OpenCode (omo) | Kimi 계열 | 코딩·도구 호출 |
| OpenClaw (2호기) | DeepSeek V4 Pro | 일반 추론·문서 표현 |
| antigravity | Google Gemini 계열 | 이종 벤더 교차 검증 |
| codex | GPT-5.x 계열 | 정밀 리뷰·보강 |

핵심 변화는 "워커 수"가 아니라 "워커 다양성"이다. 3-way 시절에는 무료 로컬 모델 위주였지만, 지금은 Kimi·DeepSeek·Gemini·GPT라는 네 벤더 계열이 서로 다른 각도에서 답을 내고, Anthropic 계열의 Claude가 이를 합성한다. 단일 모델의 환각·누락을 학습 데이터가 겹치지 않는 이종 모델들이 상호 보완하는 구조가 3-way 때보다 뚜렷해졌다.

교훈은 초판 결론과 같되 한 겹 더해졌다. **fan-out의 가치는 워커 개수가 아니라 관점의 독립성에서 나온다.** 체급이 맞지 않는 워커를 빼고 벤더가 겹치지 않는 워커를 넣는 편이, "무조건 많이"보다 합성 품질에 유리했다.

---

## 업데이트 내역

| 날짜 | 내용 |
|------|------|
| 2026-05-15 | 초판 작성 |
| 2026-07-06 | 4-Way 전환 반영 (Hermes 제외, antigravity·codex 추가) 및 보안 스크럽 |

---

*관련 글:*
- [멀티 에이전트 오케스트레이션](/development/multi-agent-orchestration/)
- [에이전트 운영 규약 설계](/development/harness-engineering-guidelines/)
- [ATH 고도화](/development/ath-advanced-features/)
