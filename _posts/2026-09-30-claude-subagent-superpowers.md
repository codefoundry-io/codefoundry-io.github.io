---
title: "Claude Code 비용 절감과 개발 속도를 위한 서브에이전트 프리셋"
categories: [AI, Claude Code]
tags: [claude-code, subagent, effort, superpowers, code-review, triad-dispatch]
description: "Claude Code 서브에이전트는 effort를 모두가 공유합니다. 즉 프리셋이 없고 본인 메인 에이전트가 xhigh로 설정되면 모든 서브에이전트가 그 기준으로 수행됩니다. 이게 낭비라고 생각돼서 effort별 비용·시간 차이, 모델×effort 프리셋 7종, Superpowers에 연결하는 배선까지 정리했습니다."
image:
  path: cover.png
  alt: "흰 배경에 주황색 도트 캐릭터가 메인 에이전트로 서 있고, 그 아래 sonnet · medium, sonnet · high, opus · xhigh 모자를 쓴 작은 서브에이전트 캐릭터 셋이 줄지어 있는 표지"
toc: true
comments: true
math: true
mermaid: true
pin: false
date: 2026-09-30 21:53:36 +0900
media_subpath: /assets/img/posts/2026-09-30-claude-subagent-superpowers/
---


Claude Code를 쓰다 보니 생각보다 느리고 토큰 소모량이 많았다.

그래서 개인적으로 알아보니 주요 원인은 Claude Code의 서브에이전트 스폰 방법이었다.
Claude Code에서는 메인 에이전트가 일부 작업을 **서브에이전트**에게 나눠 주는데, 서브에이전트는 "얼마나 깊게 생각할지"를 정하는 **effort** 값을 메인에게서 그대로 물려받는다.
그래서 메인을 가장 높은 단계인 `xhigh`로 두고 쓰면, 파일 몇 개 검색하는 서브에이전트도 `xhigh`로 돈다.
검색에 깊은 생각은 필요 없는데 시간과 비용은 깊은 생각만큼 나간다.

이 글은 그 낭비를 막으려고 서브에이전트 프리셋 7종을 만들어 공개하면서 "이게 왜 좋은지"를 설명한 내용을 정리한 것이다.
effort가 비용과 시간을 얼마나 낭비하는지, 프리셋을 나눈 근거, 그리고 Superpowers에 연결하는 방법까지 다룬다.

> Claude Code CLI 기준이며, 2026년 9월 말 공식 문서와 Superpowers v6.4.2(2026-09-25)를 기준으로 작성했습니다.
> 프리셋 7종과 CLAUDE.md 조각은 [글 끝](#presets-download)에서 내려받을 수 있습니다.
{: .prompt-info }

## 메인 에이전트와 서브에이전트 {#main-vs-subagent}

> 서브에이전트는 자기 컨텍스트에서 따로 일하고, 메인에는 결과 요약만 돌려줍니다.
{: .prompt-tip }

Claude Code에서 우리와 대화하는 상대가 **메인 에이전트**다.
메인 에이전트는 일을 직접 하기도 하고, 일부를 **서브에이전트**에게 맡기기도 한다.

```mermaid
flowchart LR
  U[사용자] --> M[메인 에이전트<br>계획 · 조율]
  M -->|작업 지시| S1[서브에이전트<br>코드 탐색]
  M -->|작업 지시| S2[서브에이전트<br>구현]
  M -->|작업 지시| S3[서브에이전트<br>리뷰]
  S1 -->|요약만 반환| M
  S2 -->|요약만 반환| M
  S3 -->|요약만 반환| M
```

서브에이전트는 AI가 등장한 후에도 조금 지나서 등장한 개념으로 대부분은 그냥 메인 에이전트로 작업을 할 것이다.
대부분 서브에이전트를 쓰는 이유는 두 가지다.

- **컨텍스트 분리**: 테스트 로그나 검색 결과처럼 양이 많은 출력은 서브에이전트 안에서 소비되고, 메인에는 요약만 돌아온다. 이 방법으로 메인 컨텍스트를 아낄 수 있다.
- **역할별 설정**: 서브에이전트마다 모델, 도구, effort를 따로 정할 수 있다. 이게 비용에 직결되는데 글의 주제가 바로 이 부분이다.

Claude Code에는 기본 서브에이전트가 세 개 들어 있지만 별 소용이 없다.
이 설정은 외부에 구독제로 CLI를 사용하는 사람들에게 거의 차이가 없지만, 비즈니스 요금제를 쓰면 너무 낭비가 크다. 다만 대부분의 사용자들은 구독 시스템을 쓰기 때문에
거의 알려지지 않은 문제점이다.

| 서브에이전트 | 용도 | 모델 |
|:--|:--|:--|
| Explore | 파일 탐색, 코드 검색 (읽기 전용) | 메인 모델 상속 |
| Plan | 플랜 모드에서 코드베이스 조사 (읽기 전용) | 메인 모델 상속 |
| general-purpose | 조사, 여러 단계 작업, 코드 수정 | `CLAUDE_CODE_SUBAGENT_MODEL` 또는 메인 모델 상속 |

생각해보면 이상하다. 모든 읽기 그리고 아웃풋은 모델의 token 단가로 책정되는데
그냥 읽기 · 요약 · grep 같은 단순한 작업도 내가 선택한 메인 에이전트 비용으로 지불해야 하는가?

Claude에서 서브에이전트는 Markdown 파일 하나로 정의할 수 있고 모델과 effort를 미리 정의하고 사용할 수 있다.
위쪽 frontmatter에 이름, 설명, 모델, effort를 적고, 본문에는 시스템 프롬프트를 적는다. 본문은 비워두고 그때그때 메인 에이전트가 선택하도록 하는 게 범용성이 높아진다.

## effort는 메인에서 상속된다 {#effort-inheritance}

> 서브에이전트 frontmatter에 `effort`가 없으면, 메인 세션의 effort를 그대로 물려받습니다.
{: .prompt-tip }

### effort란

effort는 Claude가 답을 내는 데 **토큰을 얼마나 쓸지(Thinking)** 정하는 값이다.
`low`, `medium`, `high`, `xhigh`, `max` 다섯 단계가 있다.

중요한 점은 effort가 <mark>토큰 단가를 바꾸지 않는다</mark>는 것이다.
단가는 모델별로 고정이고, effort가 높을수록 더 오래 생각한다. 이 Thinking은 output token에 해당하는 비중 높은 비용을 차지한다. 또한 생각할수록 도구를 더 많이 부르고, 출력도 길어진다.
그래서 비용과 시간이 **같이** 늘어난다.
그런데 effort가 높아질수록 결과가 좋아지느냐?
모델별 가격 차이에 따라서 한 번에 성공하면 가성비가 좋고 여러 번 실패해도 그냥 재수행으로 가성비가 높아지는 경계가 분명히 존재한다.
이걸 객관적 수치로 나열해보고 프리셋을 준비한다면 분명 같은 가격으로 좋은 성능을 낼 수 있을 것이다.

### effort 설정하기

CLI에서 `/model`을 입력하면 모델 선택 화면이 열린다.
모델을 고른 뒤 <kbd>←</kbd> / <kbd>→</kbd> 방향키로 effort를 조절하고 <kbd>Enter</kbd>를 누르면 된다.

```console
> /model
```

아래는 서브에이전트 실행 예시다. 메인 세션에서 "서브에이전트로 조사해 줘"라고 하자, 메인이 프롬프트를 써서 서브에이전트(`Agent`)를 백그라운드로 띄웠다. 모델을 따로 정하지 않았기 때문에 서브에이전트 이름 옆에 메인과 같은 `Opus 5.5 (1M context)`가 찍혀 있다.

![Claude Code에서 서브에이전트로 Claude 계열 코딩 벤치마크를 조사해 달라고 요청한 화면. Agent(Claude coding benchmark research)가 Opus 5.5 (1M context)로 백그라운드 실행되고, 메인이 작성한 조사 프롬프트가 아래에 표시된다](image.png){: w="3708" h="1066" .shadow }
_서브에이전트 실행 예시. 모델을 지정하지 않아 메인과 같은 Opus 5.5로 실행됐다._

이렇게 정한 메인 세션의 effort를 서브에이전트가 그대로 물려받는다.
공식 문서의 `effort` 필드 설명은 이렇다.[^effort-field]

> Effort level when this subagent is active. Overrides the session effort level. Default: inherits from session.

정리하면 이렇다.

```mermaid
flowchart TD
  A{서브에이전트 frontmatter에<br>effort가 있나?} -->|있음| B[그 값으로 실행]
  A -->|없음| C[메인 세션 effort로 실행]
  C --> D[메인이 xhigh면<br>검색 · 단순 구현도 xhigh]
```

> Haiku는 예외입니다. Haiku 4.5는 effort를 지원하지 않는 모델이라, 메인의 effort를 물려받아도 아무 효과가 없습니다.
{: .prompt-info }

한 가지 더.
서브에이전트를 부를 때 `model`은 호출할 때마다 바꿀 수 있지만, effort를 바꾸는 방법은 **frontmatter**뿐이다. (미리 정의한 프리셋)
그래서 "이 일은 이 모델, 이 effort로"를 고정하려면 서브에이전트 정의 파일, 즉 **프리셋**이 필요하다.

## effort에 따라 비용과 시간이 얼마나 달라질까 {#effort-cost-time}

> 같은 모델이라도 effort에 따라 토큰 사용량이 몇 배씩 달라지고, 시간도 같이 늘어납니다.
{: .prompt-tip }

### 단가

먼저 모델별 단가다. (1M 토큰당 USD. Thinking token은 출력에 해당하는 가격으로 책정)

| 모델 | 입력 | 출력 |
|:--|--:|--:|
| Fable 5.1 | $10 | $50 |
| Opus 5.5 | $4 | $20 |
| Sonnet 5.5 | $2 | $10 |
| Haiku 4.5 | $1 | $5 |

### Opus 5.5의 effort별 토큰과 시간

[Artificial Analysis](https://artificialanalysis.ai/models/claude-opus-5-5-medium)가 같은 평가 세트를 effort별로 돌린 결과다.
출력 비용은 출력 토큰 수에 출력 단가 $20을 곱해 계산했다.

표의 "지수"는 Artificial Analysis의 **Intelligence Index**(v4.3.2) 점수다.
Terminal-Bench 4.0, GDPval-AA, Humanity's Last Exam 등 10개 평가를 에이전트 · 코딩 · 일반 · 과학 추론 네 범주로 묶어 가중 평균한 0~100점 값이다.
"출력 토큰"과 "출력 비용"은 이 10개 평가를 한 벌 다 돌리는 데 든 양이다.
같은 모델을 effort만 바꿔 다섯 번 돌린 결과가 공개돼 있어서, effort가 토큰과 시간을 얼마나 바꾸는지 비교하기 좋다.
수치는 2026-10-01 모델 페이지 기준이다. 첫 응답 시간은 측정할 때마다 조금씩 바뀌는 실측값이다.
다만 종합 점수라서 "이 버그를 고칠 수 있나" 같은 특정 작업의 성공 여부까지 말해 주지는 않는다.

| effort | 지수 | 출력 토큰 | 출력 비용 | xhigh 대비 | 1점 더 올리는 비용 | 첫 응답까지 |
|:--|--:|--:|--:|--:|--:|--:|
| low | 42 | 20M | $400 | 20% | — | 12.9초 |
| medium | 51 | 38M | $760 | 38% | $40 | 23.1초 |
| high | 54 | 53M | $1,060 | 53% | $100 | 38.9초 |
| xhigh | 56 | 100M | $2,000 | 100% | $470 | 133.7초 |
| max | 58 | 260M | $5,200 | 260% | $1,600 | 702.6초 |

"1점 더 올리는 비용"은 바로 아래 단계에서 지수 1점을 더 얻는 데 추가로 드는 출력 비용이다. 예를 들어 medium은 low보다 9점 높고 $360 더 들어서 1점당 $40이다.

눈여겨볼 점은 네 가지다.

- **low → medium**: 1점당 $40로 가장 싸다. 기본값이 medium인 이유가 보인다.
- **high → xhigh**: 지수는 2점 오르는데 토큰은 약 1.9배다. 1점당 비용이 high까지의 약 5배로 뛴다.
- **xhigh → max**: 지수는 2점 오르는데 토큰은 2.6배다. 1점당 $1,600, medium 구간의 40배다.
- **시간**: 출력 속도는 초당 72~92토큰으로 비슷하다. 결국 시간은 토큰 양에 비례한다. 첫 응답까지 걸리는 시간만 봐도 medium은 23초, xhigh는 134초다.

### Sonnet 5.5의 effort별 토큰과 시간

| effort | 지수 | 출력 토큰 | 출력 비용 | 1점 더 올리는 비용 | 첫 응답까지 |
|:--|--:|--:|--:|--:|--:|
| low | 36 | 23M | $230 | — | 1.2초 |
| medium | 41 | 29M | $290 | $12 | 1.3초 |
| high | 47 | 50M | $500 | $35 | 13.1초 |
| xhigh | 52 | 100M | $1,000 | $100 | 32.6초 |

Sonnet 5.5는 high까지 1점당 비용이 Opus의 low → medium 구간보다 싸다. 반대로 xhigh는 첫 응답이 33초로 늘어나는데, Opus 5.5 medium(23초)보다 늦다.

여기서 재미있는 비교가 하나 나온다.

- Sonnet 5.5 `xhigh`: 지수 52, 출력 비용 $1,000
- Opus 5.5 `medium`: 지수 51, 출력 비용 $760

<mark>싼 모델로 바꿔도 effort가 xhigh 그대로면, 비싼 모델의 medium보다 더 비쌀 수 있다.</mark>
모델만 고르고 effort를 놓치면 절약이 안 된다. Superpowers 이야기를 할 때 이 부분이 다시 나온다.

### Fable 5.1의 effort별 토큰과 시간

Fable 5.1은 가장 비싼 모델이다. 출력 비용은 출력 단가 $50으로 계산했다.

| effort | 지수 | 출력 토큰 | 출력 비용 | 1점 더 올리는 비용 | 첫 응답까지 |
|:--|--:|--:|--:|--:|--:|
| low | 47 | 33M | $1,650 | — | 4.7초 |
| medium | 49 | 44M | $2,200 | $275 | 8.4초 |
| high | 51 | 62M | $3,100 | $450 | 29.0초 |
| xhigh | 53 | 120M | $6,000 | $1,450 | 108.0초 |
| max | 53 | 190M | $9,500 | 오르지 않음 | 271.3초 |

Opus 5.5와 Fable 5.1 모두 같은 지수 버전(v4.3.2)으로 잰 값이다. (2026-10-01 모델 페이지 기준)

effort 이름이 같다고 같은 양을 생각하는 게 아니라서, 두 모델은 effort별로 순위가 바뀐다.[^effort-names]

- **low**: Fable 5.1이 47점으로 Opus 5.5(42점)보다 높다. Anthropic도 Fable 5.1 가이드에서 "Opus · Sonnet을 높은 effort로 돌릴 자리라면, Fable 5.1 low가 과제당 비용은 비슷하면서 점수는 더 높은 경우가 많다"고 적었다.[^fable-prompting]
- **medium 이상**: Opus 5.5가 앞선다. `xhigh` 기준 Fable 5.1은 지수 53에 $6,000, Opus 5.5는 지수 56에 $2,000이다.

출력 속도도 초당 48~70토큰으로 Opus 5.5보다 느리다.
**가장 비싼 모델이 모든 일에 가장 좋은 선택은 아니다.** 다만 Fable이 "못한 모델"이라는 뜻은 아니다. Anthropic은 Fable 5.1을 일반 고객이 쓸 수 있는 가장 강한 모델로 두고, Opus 5.5로 안 풀리는 일에 쓰라고 안내한다. 이 이야기는 [메인 에이전트 선택](#choosing-main)에서 다시 한다.

### 벤치마크 점수도 effort에 정비례하지 않는다

Anthropic이 Opus 5.5 출시 페이지에 공개한 점수-비용 차트의 일부다. 차트 원자료(페이지에 들어 있는 데이터)에서 옮겼고, 비용은 과제 하나를 푸는 데 든 달러다.

| 벤치마크 | low | medium | high | xhigh | max |
|:--|--:|--:|--:|--:|--:|
| Terminal-Bench 4.0 | 38.5% / $1.29 | 57.6% / $2.94 | 64.2% / $3.88 | 66.4% / $7.35 | 64.8% / $11.24 |
| FrontierCode v1.1 | 47.3% / $0.40 | 54.6% / $0.80 | 54.0% / $1.09 | 51.4% / $2.25 | 54.4% / $6.19 |

FrontierCode에서는 `medium`이 `xhigh`보다 점수가 높고, 비용은 약 1/3이다.
Terminal-Bench에서도 `max`는 `xhigh`보다 점수가 낮은데 비용은 1.5배다.
**effort를 올린다고 늘 좋아지는 게 아니다.** 일의 종류에 맞춰 골라야 한다.

출시 페이지 본문도 같은 이야기를 한다. 기본값인 `medium`에서 Opus 5.5는 FrontierCode 54.6%로, `max`로 돌린 GPT-6 Astra의 최고점(53.3%)을 비용 약 1/5로 넘는다. CursorBench 4.0에서도 `medium` 52.5%로 Fable 5.1의 `max`(51.8%)보다 높다.[^opus-page]

Sonnet 5.5도 마찬가지다. 출시 페이지 표에 FrontierCode 46.2%가 `Max`로 적혀 있고, 각주는 "Sonnet 5.5는 Max effort에서 Xhigh보다 점수가 낮다"고 밝힌다.[^sonnet-page]

> Anthropic 출시 페이지의 표는 effort를 열마다 적지 않습니다. Opus 5.5 표는 각주에 "따로 적지 않으면 전부 max effort, Terminal-Bench 4.0만 xhigh"라고 되어 있습니다. 표의 점수를 기본값(medium)에서 기대하면 안 됩니다.
{: .prompt-info }

> 위 수치는 공개 평가 세트 기준입니다. 실제 업무에서의 비율은 직접 측정해서 확인하세요.
{: .prompt-warning }

## Superpowers 소개 {#superpowers}

> Superpowers는 "스펙 → 플랜 → 구현 → 리뷰"를 스킬로 강제하는 Claude Code 플러그인입니다. 스킬이 알아서 켜지기 때문에 사용자가 따로 명령할 필요가 없습니다.
{: .prompt-tip }

[Superpowers](https://github.com/obra/superpowers)는 Jesse Vincent(Prime Radiant)가 만든 스킬 모음이다.
README의 표현을 빌리면 "코딩 에이전트를 위한 완결된 소프트웨어 개발 방법론"이고, 그 방법론을 스킬 여러 개와 "스킬을 반드시 쓰게 만드는 초기 지시"로 구현했다.
Claude Code 외에 Codex, Gemini CLI, Cursor, Antigravity 등 16개 하네스를 지원한다.

```console
> /plugin install superpowers@claude-plugins-official
```

### 동작 방식

핵심은 **에이전트가 코드부터 짜지 않게 막는 것**이다.
"이거 만들어 줘"라고 하면 바로 구현에 들어가는 대신 한 걸음 물러나서 무엇을 원하는지 묻고, 스펙을 짧은 단위로 보여주며 확인받고, 승인이 나면 구현 플랜을 쓴다.
플랜이 승인되면 태스크마다 서브에이전트를 띄워 구현하고 리뷰하는 과정을 반복한다.
README는 "에이전트가 플랜에서 벗어나지 않고 두어 시간 자율적으로 일하는 경우가 드물지 않다"고 적고 있다.

이 흐름이 자동으로 켜지는 이유는 `using-superpowers` 스킬 때문이다.
세션 시작 훅으로 주입되는 이 스킬은 "스킬이 적용될 가능성이 1%라도 있으면 반드시 그 스킬을 먼저 부르라"고 지시한다.
그래서 사용자는 스킬 이름을 몰라도 된다. "Let's build X"라고 하면 brainstorming이, "버그가 있어"라고 하면 systematic-debugging이 먼저 켜진다.

### 기본 흐름

```mermaid
flowchart LR
  B[brainstorming<br>스펙 작성 + 셀프 리뷰] --> H0([사람: 스펙 리뷰])
  H0 --> W[using-git-worktrees<br>격리 브랜치]
  W --> P[writing-plans<br>플랜 작성 + 셀프 리뷰]
  P --> H1([사람: 플랜 리뷰<br>+ 실행 방식 선택])
  H1 -->|Subagent-driven| X[태스크마다<br>구현자 + 리뷰어]
  H1 -->|Native| N[메인이 직접<br>전체 구현]
  X --> FR[최종 리뷰<br>브랜치 전체]
  N --> FR
  FR --> H2([사람: 머지 전 리뷰])
  H2 --> F[finishing-a-development-branch<br>머지 · PR · 정리]
```

| 단계 | 스킬 | 하는 일 |
|:--|:--|:--|
| 1 | `brainstorming` | 코드 쓰기 전에 켜진다. 질문으로 요구를 다듬고, 대안을 살피고,<br>설계를 절 단위로 보여주며 확인받는다. 스펙 문서를 저장한다 |
| 2 | `using-git-worktrees` | 설계 승인 후 켜진다. 새 브랜치에 격리된 작업 공간을 만들고,<br>프로젝트 셋업과 테스트 기준선을 확인한다 |
| 3 | `writing-plans` | 승인된 설계로 플랜을 쓴다.<br>태스크마다 파일 경로, 시그니처, 테스트, 검증 방법을 적는다 |
| 4 | `subagent-driven-development`<br>또는 `executing-plans` | 플랜을 실행한다. 태스크마다 새 서브에이전트를 띄우고 매번 리뷰하거나(꼼꼼함),<br>메인이 직접 다 구현하고 마지막에 한 번 리뷰한다(저렴함) |
| 5 | `test-driven-development` | 구현 중에 켜진다. 실패하는 테스트 → 실패 확인 → 최소 구현 → 통과 확인 → 커밋.<br>테스트보다 먼저 쓴 코드는 지운다 |
| 6 | `requesting-code-review` | 태스크 사이에 켜진다. 플랜과 대조해 심각도별로 이슈를 보고한다.<br>Critical은 진행을 막는다 |
| 7 | `finishing-a-development-branch` | 태스크가 끝나면 켜진다. 테스트를 확인하고<br>머지 · PR · 보관 · 폐기 중 고르게 한 뒤 워크트리를 정리한다 |

이 밖에 디버깅용 `systematic-debugging`(4단계 근본 원인 분석)과 `verification-before-completion`(정말 고쳐졌는지 확인), 리뷰를 받는 쪽의 `receiving-code-review`, 병렬 작업용 `dispatching-parallel-agents`, 그리고 세션이 이상하게 돌았을 때 트랜스크립트를 읽고 원인을 찾아 주는 `diagnosing-superpowers`가 있다.

### 사람이 개입하는 지점

Superpowers는 실행 중에는 사람에게 묻지 않는다. "계속할까요?" 같은 확인은 사용자의 시간을 낭비한다고 보고, 애매한 건 스스로 판단(Ruling)해서 기록하고 넘어간다.
대신 문서가 저장되는 두 지점, 스펙과 플랜에서 사람이 승인해야 다음으로 넘어간다.
스펙과 플랜은 서브에이전트가 아니라 메인이 직접 셀프 리뷰한다.

플랜이 승인되면 실행 방식을 고른다. Superpowers가 그 플랜에 맞는 쪽을 이유와 함께 추천한다.

- **Subagent-driven**: 태스크마다 새 구현자와 리뷰어를 띄운다. 가장 꼼꼼하지만 태스크마다 새 컨텍스트 비용이 든다.
- **Native**: 메인이 모든 태스크를 직접 구현하고, 마지막에 가장 강한 모델로 브랜치 전체를 한 번 리뷰한다. 가장 싸고 빠르다. Superpowers는 중간 티어 세션 모델로도 잘 돈다고 설명한다.

### 플랜 작성 방식

writing-plans는 코드를 다 적지 않는다(v6.4.2부터). 플랜에는 구현자가 혼자 정할 수 없는 **결정**만 기록한다.

| 스텝 종류 | 플랜에 적는 것 |
|:--|:--|
| 테스트 스텝 | 테스트 이름과 단언(assertion), 스펙이 정한 정확한 값 |
| 코드 스텝 | 정확한 시그니처, 파일 위치, 스펙이 정한 값. 본문은 구현자가 쓴다 |
| 검증 스텝 | 실행할 명령과 통과했을 때의 출력 |
| 다른 태스크 참조 | 그 태스크의 Interfaces 블록 (코드를 반복하지 않음) |

코드 본문은 시그니처와 테스트만으로 정해지지 않는 알고리즘이거나, 스펙이 문구를 못박은 경우에만 적는다.
플랜의 독자도 "맥락이 전혀 없는 엔지니어"에서 "인터페이스와 테스트만 알면 관용적인 코드를 쓸 수 있는 엔지니어"로 바뀌었다.

릴리스 노트에 따르면 Opus 5.5 같은 모델이 플랜을 쓰다가 프로젝트 전체를 구현해 버리는 문제가 있었다고 한다.
새 방식에서는 플랜 작성 시간이 1/4, 토큰이 약 1/3로 줄었고, 이렇게 쓴 플랜도 Sonnet 5에서 기존의 전체 코드 플랜과 같은 결과(결함을 심어둔 검증 9/9 통과)를 냈다.

### TDD 작동 방식

구현 단계에서는 `test-driven-development` 스킬이 켜진다. 규칙은 한 줄이다.

> NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

테스트보다 먼저 쓴 코드는 "참고용"으로도 남기지 않고 지운다. 스킬 원문은 "지운다는 건 지운다는 뜻"이라고 못박는다.

```mermaid
flowchart LR
  R[RED<br>실패하는 테스트 작성] --> VR{의도한 이유로<br>실패하나?}
  VR -->|아니오| R
  VR -->|예| G[GREEN<br>통과할 최소 코드]
  G --> VG{전체 테스트<br>통과하나?}
  VG -->|아니오| G
  VG -->|예| RF[REFACTOR<br>정리]
  RF --> VG
  VG -->|다음| R
```

- **RED**: 원하는 동작을 보여주는 테스트를 하나만 쓴다. 실행해서 **실패하는 것을 직접 본다.** 스킬은 "실패하는 걸 안 봤으면 그 테스트가 맞는 걸 검사하는지 모른다"고 말한다.
- **GREEN**: 그 테스트를 통과시킬 최소한의 코드만 쓴다. 미리 일반화하지 않는다.
- **REFACTOR**: 테스트가 초록인 상태를 유지하며 정리하고 커밋한다.
- **초록의 기준은 프로젝트 전체 테스트다.** 태스크가 파일 하나를 지목해도 그 파일만 돌리지 않는다. v6.4.1 릴리스 노트에 따르면 12번 중 11번은 지목된 파일만 돌려서 옆 파일이 깨진 걸 못 봤고, 그래서 "프로젝트 테스트 명령을 돌리고 내가 안 만든 실패까지 이름을 대라"로 바뀌었다.

이 규칙이 프리셋과 연결된다. 플랜에 테스트 단언이 적혀 있고 구현자가 RED부터 시작하면, 싼 모델이 틀려도 테스트가 그 자리에서 잡는다. `tier-cheap`을 구현자로 쓸 수 있는 근거가 바로 이것이다.

## Superpowers의 서브에이전트 운영법 {#superpowers-subagents}

> 태스크마다 새 구현자를 띄우고, 태스크마다 리뷰하고, 마지막에 브랜치 전체를 한 번 더 리뷰합니다.
{: .prompt-tip }

`subagent-driven-development` 스킬의 핵심은 세 가지다.

1. **태스크마다 새 서브에이전트**: 구현자는 이전 태스크의 대화를 물려받지 않는다. 메인이 필요한 맥락만 골라서 넘긴다.
2. **태스크 리뷰**: 구현이 끝나면 리뷰어가 스펙 준수와 코드 품질을 확인한다. 문제가 있으면 구현자가 고치고 다시 리뷰한다.
3. **최종 리뷰**: 모든 태스크가 끝나면 브랜치 전체를 가장 강한 모델로 리뷰한다.

### 컨텍스트가 쌓이면 왜 느려지고 비싸지나

이 방식의 이점은 리뷰만이 아니다. **컨텍스트가 쌓이지 않는다.** 왜 그게 비용과 시간이 되는지부터 보자.

Claude Code는 요청마다 대화 전체를 다시 보낸다. 공식 문서의 설명이다.[^full-conversation]

> Claude Code sends your full conversation with every request, and each time Claude uses tools it sends another request carrying that batch of tool results. (...) a one-line question in a session that has been open all day still draws usage for the whole conversation.

즉 한 세션에서 태스크를 이어서 하면 매 요청이 그때까지의 대화를 전부 싣고 간다.
Anthropic 공식 문서의 그림이 이걸 그대로 보여준다. 턴이 넘어갈수록 이전 턴의 메시지와 응답이 전부 다음 턴의 입력이 된다.

![Anthropic 문서의 컨텍스트 윈도우 다이어그램. 턴 1의 입력과 출력이 턴 2의 입력이 되고, 턴 2의 것이 다시 턴 3의 입력이 되어 컨텍스트 한도까지 쌓인다](context-window-anthropic.png){: w="1600" h="900" .shadow }
_출처: Anthropic Claude Platform 문서 [Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows). "As the conversation advances through turns, each user message and assistant response accumulates within the context window, and previous turns are preserved completely."_

계산량도 같이 는다. 트랜스포머의 자기어텐션은 입력 토큰끼리 전부 짝을 지어 계산하기 때문에, 입력을 읽는 비용이 입력 길이의 제곱에 비례한다. 원 논문 "Attention Is All You Need"(Vaswani 외, 2017)의 표 1에 층당 복잡도가 $O(n^2 \cdot d)$로 적혀 있다.[^attention] 답을 생성할 때도 토큰 하나를 낼 때마다 지금까지의 컨텍스트 전체를 참조한다. 컨텍스트가 두 배면 읽는 비용은 네 배, 생성 비용은 두 배가 되는 구조다.

![Longformer 논문 그림 1. 입력 길이(seq len)에 따른 시간(ms/batch)과 메모리(MiB). Full self-attention 파란 선이 길이가 늘수록 위로 꺾이며 메모리 축에서는 측정 범위를 벗어난다](longformer-fig1.png){: w="1600" h="693" .shadow }
_출처: Beltagy, Peters, Cohan, "Longformer: The Long-Document Transformer" (2020), Figure 1, [arXiv:2004.05150](https://arxiv.org/abs/2004.05150). 파란 선(Full self-attention)이 일반 트랜스포머로, 입력 길이가 늘수록 시간과 메모리가 제곱으로 늘다가 GPU 메모리가 바닥나 측정이 끊긴다._

```mermaid
flowchart TB
  subgraph S["한 세션에서 이어서 (컨텍스트가 쌓임)"]
    direction LR
    s1["요청 1<br>시스템 + T1"] --> s2["요청 2<br>시스템 + T1 + T2"] --> s3["요청 3<br>시스템 + T1 + T2 + T3"] --> s4["...<br>매 요청이 더 무거워짐"]
  end
  subgraph D["Subagent-driven (매번 새 컨텍스트)"]
    direction LR
    d1["구현자 1<br>시스템 + T1"]
    d2["구현자 2<br>시스템 + T2"]
    d3["구현자 3<br>시스템 + T3"]
    m["메인<br>요약 3개만 보유"]
    d1 -.요약.-> m
    d2 -.요약.-> m
    d3 -.요약.-> m
  end
```

이게 비용과 시간이 되는 경로는 세 가지다.

1. **입력 토큰은 매 요청 과금된다.** 캐시에 맞으면 싸지만 공짜는 아니다. Opus 5.5 기준 캐시 읽기는 입력 단가의 5%($0.20/MTok)다. 200k 토큰이 쌓인 세션에서 한 줄 질문을 던지면 그 200k를 매번 다시 읽는 값을 낸다.
2. **캐시는 시간이 지나면 전부 다시 처리한다.** 구독은 1시간, API 키는 5분이 지나면 캐시가 사라지고, 다음 요청은 전체 컨텍스트를 정가로 다시 처리한다. 프롬프트 캐싱 문서의 예시로 200k 문서에 50토큰 질문을 하면, 캐시 적중 시 20,050토큰 값, 미적중 시 200,050토큰 값이다. 
3. **처리 시간도 입력 길이를 따라간다.** 모델은 답을 내기 전에 입력 전체를 읽어야 한다. 프롬프트 캐싱 문서가 캐싱의 효과로 "긴 문서에서 첫 토큰까지의 시간이 개선된다"고 적는 것 자체가, 입력이 길수록 첫 응답이 늦어진다는 뜻이다.

숫자로 보면 이렇다. 태스크 하나가 대화를 40k 토큰씩 늘린다고 치자.
한 세션에서 태스크 5개를 이어서 하면 다섯 번째 요청은 200k를 싣고 가고, 다섯 요청이 읽는 입력의 합은 40k × (1+2+3+4+5) = 600k다.
Subagent-driven이면 구현자 다섯이 각각 40k씩, 합 200k다. 메인은 요약만 받으니 거의 늘지 않는다.
태스크마다 요청이 여러 번 오간다는 걸 감안하면 실제 차이는 이보다 크다.

Anthropic의 Opus 5.5 소개 글은 Claude Code 사용 패턴이 이미 이 방향이라고 말한다. 요청당 컨텍스트가 2.6배로 늘었고, 입력 대 출력 토큰 비율이 189:1에서 324:1이 됐다. 비용의 대부분이 출력이 아니라 **쌓인 입력을 다시 읽는 데** 들어간다는 뜻이다.

Subagent-driven에서는 구현자마다 새 컨텍스트로 시작하고, 이전 태스크의 대화를 물려받지 않는다. 다섯 번째 태스크도 첫 번째 태스크와 같은 크기로 돈다.
메인은 각 서브에이전트의 요약만 받으니 메인 컨텍스트도 조율에만 쓰인다. Superpowers 스킬 원문도 "서브에이전트는 세션의 컨텍스트나 히스토리를 절대 물려받지 않는다"와 "이것이 메인의 컨텍스트를 조율 작업용으로 보존한다"를 이유로 든다.
태스크가 많을수록 이 방식이 유리하다.

역할별로 모델을 고르는 규칙(Model Selection)도 있다. 원칙은 "그 역할을 해낼 수 있는 가장 약한 모델"이다.

| 역할 | Superpowers의 기준 |
|:--|:--|
| 구현 — 플랜에 코드 본문까지 있거나, 단일 파일 기계적 수정 | 가장 싼 티어 |
| 구현 — 1~2파일, 스펙이 명확한 독립 함수 | 빠르고 싼 모델 |
| 구현 — 플랜이 설명 위주 | 중간 티어 이상 |
| 구현 — 여러 파일 통합, 디버깅 | 표준 모델 |
| 구현 — 설계 판단 필요 | 가장 강한 모델 |
| 태스크 리뷰 | diff 크기와 위험도에 맞게. 최소 중간 티어 |
| 수정 후 재리뷰 | 싼 티어 ~ 중간 티어 |
| 수정 4~5회차 구현 | 막힌 구현자보다 한 티어 위 |
| 최종 리뷰 | 가장 강한 모델 (세션 기본 모델이 아니라) |

그리고 이런 조언이 붙어 있다.[^turn-count]

> Turn count beats token price. (...) the cheapest models routinely take 2-3× the turns on multi-step work — costing more overall.

가장 싼 모델은 여러 단계 작업에서 턴을 2~3배 더 쓰는 경우가 많아, 결국 더 비쌀 수 있다는 뜻이다.
무조건 제일 싼 설정이 답은 아니라는 점에서 앞의 effort 표와 같은 이야기다.

반대로 말하면, 몇 번 틀려도 이상할 게 없을 만큼 짧은 구현이나 빈칸만 채우면 되는 구현은 싼 모델이 몇 번 틀려도 손해가 아니라는 뜻이기도 하다.
Superpowers가 이미 플랜을 짜 놓은 구현이 여기에 가깝다.
v6.4.2 플랜은 코드 본문을 적지 않지만 시그니처, 테스트 단언, 스펙 값이 이미 정해져 있다. 구현자가 새로 판단할 것이 적고, 틀리면 테스트가 바로 잡아낸다.
Superpowers도 같은 기준이다. 스펙이 명확한 1~2파일짜리 독립 함수나 단일 파일의 기계적 수정에는 빠르고 싼 모델을 쓰라고 하고, 플랜이 잘 짜여 있으면 대부분의 구현 태스크가 여기에 해당한다고 말한다.

얼마나 틀려도 되는지 대략 계산해 보자. 같은 평가 세트 한 벌을 돌린 출력 비용 기준이다.

| 모델 | 출력 비용 | 같은 비용으로 Sonnet 5.5 medium을 돌릴 수 있는 횟수 |
|:--|--:|--:|
| Sonnet 5.5 medium | $290 | 1번 |
| Sonnet 5.5 xhigh | $1,000 | 약 3번 |
| Opus 5.5 xhigh | $2,000 | 약 7번 |
| Fable 5.1 xhigh | $6,000 | 약 20번 |

메인이 Opus 5.5 `xhigh`라면, 메인 모델로 한 번 구현할 비용으로 Sonnet `medium`은 **약 7번** 다시 시도할 수 있다.
틀릴 일이 거의 없는 구현이라면 싼 쪽이 남는 장사다.

## Superpowers 가이드대로 하면 어떤 모델이 불릴까 {#what-gets-called}

> Superpowers는 모델 "티어"만 정하고 이름과 effort는 정하지 않습니다. 이름은 메인이 그때그때 고르고, effort는 전부 메인을 따라갑니다.
{: .prompt-tip }

Superpowers의 서브에이전트 템플릿은 모두 `general-purpose`에 `model`을 채워 넣는 형태다.

```text
Subagent (general-purpose):
  description: "Implement Task N: [task name]"
  model: [MODEL — REQUIRED: choose per SKILL.md Model Selection; an omitted
         model silently inherits the session's most expensive one]
```
{: file="skills/subagent-driven-development/implementer-prompt.md" }

여기서 두 가지를 알 수 있다.

- 스킬에는 "싼 티어", "가장 강한 모델" 같은 **티어만** 있고 모델 이름은 없다. 실제 이름은 메인 에이전트가 Claude Code의 모델 별칭(`haiku`, `sonnet`, `opus`, `fable` 등)에서 골라 채운다.
- `general-purpose`에는 effort 설정이 없다. 그래서 **effort는 전부 메인을 따라간다.**

메인을 Opus 5.5 `xhigh`로 두고 Superpowers 가이드대로 돌리면 이렇게 된다.

| 역할 | Superpowers 기준 | 메인이 고를 모델 | 실제 effort |
|:--|:--|:--|:--|
| 스펙 · 플랜 셀프 리뷰 | 서브에이전트 없이 메인이 직접 | Opus 5.5 (메인) | xhigh |
| 단독 코드 리뷰 (requesting-code-review) | 템플릿에 `model` 줄이 없음 | Opus 5.5 (메인 그대로) | xhigh |
| 구현 — 1~2파일 독립 함수, 단일 파일 수정 | 빠르고 싼 모델 | Haiku 4.5 | 적용 안 됨 |
| 구현 — 설명 위주, 여러 파일 통합 | 중간 · 표준 티어 | Sonnet 5.5 | xhigh |
| 구현 — 설계 판단 | 가장 강한 모델 | Opus 5.5 | xhigh |
| 태스크 리뷰 | 최소 중간 티어 | Sonnet 5.5 (위험하면 Opus) | xhigh |
| 수정 후 재리뷰 | 싼 티어 ~ 중간 티어 | Haiku 또는 Sonnet | 적용 안 됨 / xhigh |
| 수정 4~5회차 구현 | 한 티어 위 | Sonnet → Opus | xhigh |
| 최종 리뷰 | 가장 강한 모델 | Opus 5.5 또는 Fable | xhigh |

<!-- TODO: 실제 세션 로그(~/.claude/projects/**/*.jsonl)에서 메인이 고른 model 값을 확인해 표 보정 -->

이 표에서 문제가 세 가지 보인다.

1. **effort가 전부 xhigh다.** Superpowers가 비용을 아끼려고 Sonnet을 골라도 Sonnet `xhigh`가 된다. 앞에서 봤듯이 이건 Opus `medium`보다 비쌀 수 있다.
2. **모델 이름이 실행마다 달라질 수 있다.** "가장 강한 모델"을 메인이 `opus`로 읽을 수도, `fable`로 읽을 수도 있다. Claude Code의 `best` 별칭도 Fable을 쓸 수 있으면 Fable, 아니면 Opus로 풀린다. Fable 5.1의 출력 단가는 $50로 Opus 5.5의 2.5배이고, 앞의 표 기준으로 최종 리뷰 한 번이 $2,000에서 $6,000이 된다.
3. **어느 티어로 갈지가 흔들린다.** v6.4.2 플랜의 태스크는 대부분 시그니처와 테스트만 적혀 있다. 이걸 "스펙이 명확한 독립 함수(싼 티어)"로 볼지 "여러 파일 통합(표준)"으로 볼지는 메인의 판단이라, 같은 플랜도 실행마다 Haiku와 Sonnet을 오갈 수 있다.

<mark>배선이 중요한 이유가 이것이다.</mark> 티어마다 "어떤 모델, 어떤 effort"를 프리셋으로 못박아 두면 세 문제가 한꺼번에 사라진다.

### 예 1. 구현자 (여러 파일 통합 태스크)

- **그대로 쓰면**: Superpowers 기준대로 Sonnet을 골라도, effort는 `xhigh`를 물려받아 Sonnet 5.5 `xhigh`로 실행된다.
- **프리셋**: Sonnet 5.5 `medium`
- **차이**: 출력 토큰 100M → 29M, **약 71% 감소**. 첫 응답까지 33초 → 1.3초.

Superpowers가 비용을 아끼려고 모델을 낮춰도, effort가 따라 내려오지 않으면 효과가 사라진다.

### 예 2. 태스크 리뷰어

- **그대로 쓰면**: 최소 중간 티어 기준으로 Sonnet을 골라도 Sonnet 5.5 `xhigh`로 실행된다.
- **프리셋**: Sonnet 5.5 `high`
- **차이**: 출력 토큰 100M → 50M, **약 50% 감소**. 첫 응답까지 33초 → 13초.

태스크 하나의 diff를 스펙과 대조하는 일이라 `high`로도 충분한 경우가 많다.

### 예 3. 태스크 5개짜리 플랜 한 번 돌리기

Superpowers가 부르는 서브에이전트를 전부 세어 보자.
Subagent-driven 방식에서 태스크가 여러 파일에 걸쳐 구현자가 Sonnet으로 가고, 각 서브에이전트가 비슷한 양의 일을 한다고 단순하게 가정했다.
스펙과 플랜의 셀프 리뷰는 메인이 직접 하므로 표에서 뺐다.

| 역할 | 개수 | 그대로 쓰면 | 비용 | 프리셋 | 비용 |
|:--|--:|:--|--:|:--|--:|
| 구현자 | 5 | Sonnet xhigh | $5,000 | Sonnet medium | $1,450 |
| 태스크 리뷰어 | 5 | Sonnet xhigh | $5,000 | Sonnet high | $2,500 |
| 최종 리뷰어 | 1 | Opus xhigh | $2,000 | Opus xhigh (유지) | $2,000 |
| **합계** | 11 | | **$12,000** | | **$5,950** |

비용이 **약 50%** 줄어든다. 최종 리뷰는 일부러 `xhigh`로 남겨뒀다.
프리셋의 목적은 무조건 낮추는 게 아니라, **아낄 곳에서 아끼고 써야 할 곳에 쓰는 것**이다.

> 금액은 공개 평가 세트 한 벌을 돌린 비용 기준입니다. 실제 금액이 아니라 비율로 보세요.
{: .prompt-warning }

### 예 4. 구성별 비용 예상치

같은 태스크 5개짜리 플랜을 네 가지 구성으로 돌렸을 때 서브에이전트(또는 메인이 직접 한 구현) 비용을 비교한 표다.
단위는 앞과 같이 "평가 세트 한 벌"이고, 메인이 조율에 쓰는 비용은 네 구성 모두 비슷하다고 보고 뺐다.
메인 effort가 `high`인 경우와 `xhigh`인 경우를 나눴다. 상속 때문에 이 둘의 차이가 크다.

| 구성 | 구현 5개 | 태스크 리뷰 5개 | 최종 리뷰 | 메인 high | 메인 xhigh |
|:--|:--|:--|:--|--:|--:|
| A. Superpowers 없이<br>메인이 직접 구현, 리뷰 없음 | Opus (메인 effort) | 없음 | 없음 | $5,300 | $10,000 |
| B. Superpowers 그대로<br>(Subagent-driven) | Sonnet (메인 effort 상속) | Sonnet (상속) | Opus (상속) | $6,060 | $12,000 |
| C. 프리셋만 두고 배선 없음 | B와 같음 | B와 같음 | B와 같음 | ≈ $6,060 | ≈ $12,000 |
| D. 프리셋 + 배선 | `tier-cheap` | `tier-standard` | `tier-max` | **$5,950** | **$5,950** |

- **A**: 리뷰가 없어서 싸 보이지만, 구현이 전부 메인 모델(Opus)로 돈다. 메인이 xhigh면 D의 1.7배다.
- **B**: Superpowers가 모델은 Sonnet으로 낮췄지만 effort가 메인을 따라간다. 메인이 xhigh면 구현자와 리뷰어 열 개가 전부 Sonnet xhigh($1,000)로 돌아 D의 2배가 된다.
- **C**: 프리셋 파일이 있어도 Superpowers 템플릿은 `general-purpose`를 이름으로 부르고 `model`을 직접 넘긴다. 배선이 없으면 프리셋은 그냥 지나친다. 메인이 가끔 description을 보고 프리셋을 고를 수는 있어서 "≈"로 적었다.
- **D**: 메인 effort와 상관없이 같은 비용이다. 메인이 high일 때는 B와 비용이 거의 같지만, 그 안에서 구현 비용을 줄이고 최종 리뷰를 xhigh로 올렸다. 메인이 xhigh일 때는 절반이다.

정리하면, 프리셋과 배선의 효과는 <mark>메인 effort가 높을수록 커진다.</mark> 메인이 high면 "같은 돈으로 최종 리뷰를 더 강하게", 메인이 xhigh면 "비용 절반"이다.

> A~D 모두 서브에이전트 하나가 비슷한 양의 일을 한다는 단순 가정입니다. 실제로는 Superpowers가 경고한 대로 싼 모델이 턴을 더 쓸 수 있고, 리뷰에서 문제가 나오면 수정 · 재리뷰 비용이 추가됩니다.
{: .prompt-warning }

## 메인 에이전트는 무엇으로 할까 {#choosing-main}

> 메인은 Opus 5.5가 기본입니다. 목표가 뚜렷하면 `/goal`을 붙이고, Opus로 막히는 어려운 문제만 Fable 5.1로 올립니다. Anthropic 공식 가이드가 권하는 순서도 같습니다.
{: .prompt-tip }

서브에이전트 프리셋을 정하기 전에 메인부터 정해야 한다.
메인은 사람과 대화하고, 플랜을 쓰고, 서브에이전트를 부르고 결과를 합친다.

비교는 Anthropic 공식 자료(출시 페이지, 시스템 카드, 플랫폼 문서)를 중심으로 하고, 벤치마크 운영처가 직접 공개한 값만 더했다. 모델마다 다른 출처나 다른 벤치마크 버전을 섞지 않도록, 여러 모델을 한 표에 놓을 때는 한 출처에서 가져왔다. 여러 출처의 점수를 옮겨 모은 집계 사이트, 고객 인용, 한두 번 돌린 개인 실험은 근거에서 뺐다.

이 글은 **Claude Code CLI로 코딩하는 상황**이 기준이다. 벤치마크는 돌린 하네스(에이전트 실행 환경)에 따라 결과가 달라지니, 하네스도 같이 적었다.

| 벤치마크 | 하네스 | Claude Code 기준 |
|:--|:--|:--:|
| Terminal-Bench 4.0 · Science 0.1 | Claude Code (`--bare` 모드) | ✅ |
| FrontierCode (Cognition 측정) | Claude Code | ✅ |
| CursorBench | Cursor 자체 에이전트 | ❌ |
| FrontierSWE v2 | Proximal 자체 하네스 | ❌ |
| AA-Briefcase · GDPval-AA · AA 지수 | Artificial Analysis 하네스 | ❌ |
| MCP Atlas | Scale AI 하네스 | ❌ |
| SWE-bench · DeepSWE | 시스템 카드에 명시 없음 | ? |

<mark>Claude Code로 잰 코딩 벤치마크는 Terminal-Bench와 FrontierCode 둘이다.</mark> 둘 다 아래에서 effort별로 본다. 나머지는 참고로 읽는다.[^opus-sc][^fable-sc]

### 먼저, Anthropic은 두 모델을 어떻게 두나

벤치마크를 보기 전에 공식 위치부터 짚어 둔다. 벤치마크만 보면 Fable 5.1이 Opus 5.5보다 못해 보이는데, 공식 문서는 그렇게 말하지 않는다.

- **Fable 5.1**: "Anthropic's most capable model open to all customers." 일반 고객이 쓸 수 있는 가장 강한 모델이다.[^choosing]
- **Opus 5.5**: 대부분의 작업을 여기서 시작하라고 한다. 모델 선택 가이드의 순서는 "Opus 5.5로 구현 → effort를 xhigh · max까지 올려도 어려운 추론이나 장기 에이전트 작업이 모자라면 Fable 5.1로 이동"이다.[^choosing]
- **출시 페이지의 단서**: Opus 5.5 출시 페이지는 도입부에서 Opus 5.5가 "대부분의 작업에서 Fable 5.1 수준"이라고 했고, 벤치마크 절에서는 벤치마크 차이를 그대로 믿지 말라고 적었다.[^opus-page]

> That said, at these levels of capability we've found that benchmark margins have become a less reliable guide to real-world differences. In our own use, the gap between Opus 5.5 and Claude Fable 5.1 is narrower than these scores suggest.

Opus 5.5는 Fable 5.1보다 3주 늦게 나온 다음 세대(5.5) 모델이다. 그래서 벤치마크에서는 따라잡거나 앞서지만, Anthropic은 Fable을 여전히 **Opus 위에 한 단계 더 있는 에스컬레이션 모델**로 둔다. 아래 숫자는 이 틀 안에서 읽어야 한다.

### 1. 터미널에서 여러 단계 작업을 얼마나 잘하나

Terminal-Bench는 두 분야가 있다. **4.0**은 명령줄 안에서 여러 단계로 이어지는 소프트웨어 작업, **Science 0.1**은 터미널에서 하는 과학 연구 작업이다.

출시 페이지 표는 모델마다 가장 잘 나온 effort의 점수만 싣는다. 그래서 effort별 차트 원자료로 같은 effort끼리 놓았다. Claude Code `--bare` 모드로 잰 값이다.[^opus-page][^sonnet-page]

| Terminal-Bench 4.0 | low | medium | high | xhigh | max |
|:--|--:|--:|--:|--:|--:|
| Sonnet 5.5 | 20.0% | 28.8% | 43.0% | 61.5% | **70.6%** |
| Opus 5.5 | 38.5% | **57.6%** | **64.2%** | **66.4%** | 64.8% |
| Fable 5.1 | **40.2%** | 43.4% | 49.4% | 51.3% | 55.8% |

- **최고점**만 보면 Sonnet 5.5 > Opus 5.5 > Fable 5.1이다. 다만 Sonnet의 70.6%는 `max`에서 나오고, 과제당 $12.54로 Opus 5.5 `xhigh`($7.35)보다 비싸다.
- **medium ~ xhigh**에서는 Opus 5.5가 가장 높다.
- **low**에서는 Fable 5.1(40.2%)이 Opus 5.5(38.5%)보다 조금 높지만 표준 오차 안이다.
- Science 0.1은 Opus 5.5 58.7%, Fable 5.1 52.6%다(max).

Opus 5.5 출시 페이지가 밝힌 표준 오차는 Opus 5.5 ±2.6pt, 다른 Claude 모델 ±1.6~2pt다. Sonnet 5.5의 표준 오차는 공개되지 않았다. 최고점끼리의 차이(4.2pt)는 크지 않은 편이다.

### 2. 긴 작업을 끊기지 않고 이어가나

작업 연속성은 두 가지로 나눠 봐야 한다. **오래 일관되게 판단하는가**(벤치마크가 재는 것)와 **사람 없이 혼자 계속 가는가**(무인 실행에서 멈추지 않는가)다. 결론부터 말하면 앞의 것은 공식 벤치마크 전부에서 Opus 5.5가 Fable 5.1보다 앞서고, 뒤의 것은 두 모델 모두 공식 문서가 같은 약점을 인정한다. 다만 앞의 벤치마크들은 Claude Code가 아닌 하네스로 쟀다.

**오래 일관되게 판단하는가.** 몇 시간짜리 과제를 재는 벤치마크를 시스템 카드에서 모았다.[^opus-sc][^fable-sc] 셋 다 Claude Code 하네스가 아니다.

| 벤치마크 | 재는 것 | Opus 5.5 | Fable 5.1 |
|:--|:--|--:|--:|
| FrontierSWE v2 (Proximal) | 최상위 모델이 과제당 20시간 가까이<br>일하기도 하는 엔지니어링 · 연구 34개 | **62.3%** | 56.3% |
| DeepSWE v1.1 | 긴 호흡의 소프트웨어 과제 113개 | **74.2%** | 67.4% |
| AA-Briefcase v1.1 | 긴 호흡의 지식 노동 (Elo) | **1822** | 1678 |

- 모두 Opus 5.5 시스템 카드 값이다. AA-Briefcase는 8.1 요약 표, FrontierSWE v2는 8.7절(Proximal 측정), DeepSWE는 8.3절이다. Fable 5.1의 DeepSWE만 Fable 시스템 카드 값이다. (Fable 시스템 카드는 FrontierSWE v2를 0.57로 적었다.)
- FrontierSWE v2 전체 1위는 GPT-6 Astra(65.5%)이고, Opus 5.5는 2위다.
- Sonnet 5.5의 AA-Briefcase는 1811로 Opus 5.5(1822)와 11점 차이다. Anthropic은 Sonnet 쪽 측정이 버그가 있던 사전 배포판에서 이뤄져 점수가 약간 낮게 나왔을 수 있다고 적었다.[^sonnet-page] Sonnet 5.5의 FrontierSWE · DeepSWE 점수는 공개되지 않았다.
- Fable 5.1이 나왔을 때는 FrontierSWE v2 1위였다. 시스템 카드는 Opus 5와 달리 과제 난도가 올라가도 성능이 유지되고, 비교한 Claude 세 모델(Fable 5.1 · Opus 5 · Fable 5) 중 완전 실패율이 가장 낮다(5%)고 적었다.[^fable-sc] 3주 뒤 나온 Opus 5.5가 그 점수를 넘었다.

<details markdown="1">
<summary>근거 더 보기 — Anthropic 내부 테스트 · METR · Vending-Bench 2</summary>

**Anthropic 내부 테스트** (Opus 5.5 출시 페이지): HAProxy를 C에서 Rust로 옮기는 작업을 두 모델에 맡겼다. 둘 다 HAProxy 자체 회귀 테스트를 거의 전부 통과했고, Opus 5.5는 9.5시간, Fable 5.1은 12시간 걸렸다. 비용은 Opus 5.5가 51% 적었다.[^opus-page] Anthropic이 직접 돌리고 발표한 한 건이라는 점은 감안해야 한다.

**METR** (Opus 5.5 시스템 카드에 실린 외부 평가, AI R&D 역량 중심): AI R&D 역량에서 Opus 5.5는 Fable 5.1보다 "a modest improvement", 즉 점진적으로 나아진 수준이라고 평가했다. 연구자의 판단력이나 "taste" 같은 능력에서는 Fable 5.1보다 크게 나아졌다는 근거가 없다고 적었다.[^opus-sc]

**Vending-Bench 2** (Andon Labs 공식 리더보드): 1년치 자판기 사업을 운영하며 최종 잔고로 점수를 매긴다.

| 모델 | 순위 | 최종 잔고 |
|:--|--:|--:|
| Opus 5 | 4위 | $11,182 ± $2,094 |
| Opus 4.7 | 5위 | $10,937 ± $1,181 |
| Opus 5.5 | 8위 | $9,235 ± $785 |

Opus 5.5의 평균은 전작 Opus 5보다 낮지만, 두 모델의 오차 범위가 겹친다. Fable 5.1도 리더보드 차트에는 있지만 공식 페이지 본문에서 수치를 확인하지 못해 표에서 뺐다.

</details>

**혼자 계속 가는가.** Claude Code에서 써 보면 Opus 5.5는 자꾸 멈춰서 묻는다. 이건 착각이 아니다. Anthropic의 Opus 5.5 프롬프트 가이드가 "무인 에이전트 실행" 절에서 이 현상을 그대로 인정한다.[^unattended]

> On long tasks with several parts, Claude Opus 5.5 keeps the user updated as it works, and some of those updates end the turn with text rather than a tool call. An unattended agent loop that treats such a turn as the end of the task stops running there.

가이드는 Opus 5.5가 일이 남았는데도 턴을 끝내는 네 가지 방식을 꼽는다. 한 일을 길게 요약하고 "다음은 이걸 하겠다"로 끝내기, "원하시면 계속하겠다"고 묻기, 막지도 않는 결정 목록을 사용자에게 넘기기, 마일스톤이 끝났으니 보고할 때라고 판단하기.

그런데 이건 Opus만의 버릇이 아니다. Fable 5.1 프롬프트 가이드도 "Finish the whole task" 절에서 같은 현상을 적었다.[^fable-prompting]

> Without the nudge, the model sometimes describes what it would do next instead of doing it ("Next, I'll …") or stops to ask permission for a step the original request already covered ("Shall I apply this?").

같은 절은 Fable 5.1이 "목표가 분명하면 방법론 지시 없이도 아주 긴 작업을 해낸다"고 하면서도, 복잡한 비동기 작업에서는 턴을 일찍 끝내지 말라는 지시문을 넣으라고 한다. 두 가이드 모두 시스템 프롬프트에 넣을 지시문을 준다. Opus 5.5 쪽 요지는 "도구 호출이 없는 메시지는 턴을 끝내니, 상태 보고와 결정 제안은 다음 도구 호출과 같은 메시지에 넣고 사용자 답이 필요 없는 일은 계속하라. 멈춰도 되는 건 사용자 없이는 아무것도 못 움직일 때뿐이다"이다. Fable 5.1 쪽 요지는 "사용자는 실시간으로 보고 있지 않다. 되돌릴 수 있는 일은 묻지 말고 진행하고, 턴을 끝내기 전에 마지막 문단이 계획이나 약속이면 그 일을 지금 하라"이다.

정리하면 체감으로는 Opus가 더 자주 멈추지만, 공식 문서 기준으로 <mark>두 모델 모두 무인 실행에는 지시문이 필요하다.</mark> 사람 없이 돌리는 세션에는 해당 모델 가이드의 지시문을 넣고, 페어 코딩처럼 사람이 옆에서 답하는 세션에는 넣지 않는다. Opus 5.5 가이드가 사람이 함께하는(human-in-the-loop) 작업에서는 빼라고 적었다. Superpowers도 스킬 안에 "태스크 사이에 사용자에게 확인하지 말고 끝까지 실행하라"는 지시가 있다.

Fable 5.1 가이드에는 주의할 점이 하나 더 있다. 열린 기능 구현을 맡기면 "요청한 것보다 더" 할 때가 있다고 한다. 근처 코드를 고치거나, 요청에 없는 동작을 넣거나, 테스트 파일을 필요 이상으로 커밋한다.[^fable-prompting] FrontierCode에서 Fable 5.1의 점수가 effort를 올릴수록 깎이는 이유도 같다(아래 [코딩 · 설계 벤치마크](#coding-design-benchmarks)에서 다룬다). Fable을 메인에 둔다면 리뷰에서 범위 밖 수정을 보는 게 좋다.

### 3. 서브에이전트를 얼마나 잘 다루나

"서브에이전트를 조율하는 능력"에 가장 가까운 공개 벤치마크는 **MCP Atlas**(Scale AI)다.
여러 MCP 서버에 흩어진 도구를 찾아서, 맞는 인자로 부르고, 실패하면 복구하고, 결과를 합쳐 답을 내는 여러 단계 도구 조율을 잰다. 서브에이전트를 부르는 것도 결국 Agent라는 도구를 조율하는 일이라, 이 능력과 가장 가깝다.

Scale AI 공식 리더보드의 값이다.[^mcp-atlas]

| 모델 | MCP Atlas |
|:--|--:|
| Muse Spark 1.1 (공동 1위) | 88.10 ± 1.95 |
| Fable 5.1 (공동 1위) | 87.20 ± 2.05 |
| Opus 5 (xhigh, 2위) | 85.80 ± 2.10 |
| Fable 5 | 83.30 ± 2.25 |
| Opus 4.8 (max) | 82.20 ± 2.40 |
| Opus 5.5, Sonnet 5.5 | 미측정 |

Scale은 오차 범위가 겹치는 모델에 같은 순위를 준다. Fable 5.1은 전체 공동 1위이고 Claude 모델 중 가장 높다. 다만 Opus 5와의 차이(1.4pt)도 오차 범위(±2) 안이고, Opus 5.5와 Sonnet 5.5는 아직 점수가 없다. 이 항목은 **판단 보류**가 맞다.

<details markdown="1">
<summary>더 보기 — 시스템 카드의 멀티 에이전트 평가 · 학계 벤치마크</summary>

Fable 5.1 시스템 카드에는 멀티 에이전트 평가가 있다. 바이너리와 문서만 보고 프로그램을 다시 만드는 ProgramBench에서 Fable 5.1 하나로 푸는 것과 여러 에이전트로 푸는 것을 비교했다. 다섯 에이전트 팀은 같은 점수(0.6)에 두 배 빨리 도달했고, 필요할 때 서브에이전트를 띄우는 구성은 최종 점수가 가장 높았다. 대신 토큰은 더 쓴다.[^fable-sc] 같은 평가를 Opus 5.5로 한 결과는 공개되지 않았다.

학계에도 오케스트레이션 전용 벤치마크가 있다. OrchestraBench(위임 실패 유형과 복구, 분해 품질)와 OrchBench(오케스트레이션 플랜을 시뮬레이터로 채점)인데, 둘 다 Claude 5.x 점수는 없다. OrchestraBench의 Sonnet 4.6 · Opus 4.8 · Haiku 4.5 결과에서는 세 모델 모두 "모호한 위임"과 "컨텍스트 오염"을 거의 복구하지 못했다. 모델 급을 올려도 조율 실패는 남는다는 뜻이라, 배선처럼 사람이 규칙을 못박는 이유가 된다.

</details>

Anthropic의 비용 · 지능 최적화 가이드에는 실측한 조합 예시가 있다.[^cost-intel] 하네스는 Claude Code가 아니라 API 에이전트다.

- **조율자 + 워커**: 대형 코드베이스 결함 찾기에서 Fable 5.1 리드 + Sonnet 5 워커 25개가 단독 실행 대비 비용은 절반, 점수는 10~12점 낮았다. 최고 정확도는 여전히 Fable 5.1 단독 `high`였다.
- **어드바이저 + 실행자**: 코딩 벤치마크에서 Opus 5.5 `high` 실행자에 Fable 5.1 어드바이저를 붙이면 Opus 5.5 단독 `high`보다 1.7점 높았다. 실행 간 편차 수준의 차이에 비용은 약 2.1배였다.

두 예시 모두 조율자 · 어드바이저 자리에 Fable 5.1을 썼다. 다만 Opus 5.5와 Fable 5.1 중 누가 조율을 더 잘하는지 직접 비교한 자료는 아니다. Managed Agents의 멀티 에이전트 예시 코드는 조율자에 `claude-opus-5-5`, 리뷰어에 `claude-haiku-4-5`를 쓴다. 두 모델의 프롬프트 가이드에는 멀티 에이전트용 팁이 각각 있다. Opus 5.5는 경과 시간을 알려 주면 서브에이전트를 더 병렬로 돌려 빨리 끝내고, Fable 5.1은 서브에이전트가 도는 동안 리드가 기다리지 않게 하면 완료 시간이 줄어든다.[^unattended][^fable-prompting] 어느 쪽이 조율을 더 잘한다는 비교는 없다.

### 판단

| 기준 | Sonnet 5.5 | Opus 5.5 | Fable 5.1 |
|:--|:--|:--|:--|
| 공식 위치 | 잘 정의된 일상 작업 | 대부분 작업의 시작점 | Opus로 모자랄 때 올리는<br>가장 강한 모델 |
| 터미널 작업 (TB 4.0, Claude Code) | 최고점 1위<br>(max 70.6%, 과제당 $12.54) | medium ~ xhigh에서 1위 | low에서 Opus와 비슷<br>(오차 범위) |
| 오래 일관되게 판단 (시스템 카드) | Briefcase 1811<br>(나머지 자료 없음) | 셋 다 Fable 5.1보다 높음 | 셋 다 Opus 5.5보다 낮음 |
| 혼자 계속 감 (공식 가이드) | 자료 없음 | 텍스트로 턴을 끝내고 멈춤.<br>지시문 제공 | 같은 버릇 있음. 지시문 제공.<br>범위 밖 수정 주의 |
| 도구 · 서브에이전트 조율 (MCP Atlas) | 미측정 | 미측정 (Opus 5는 85.8) | Claude 중 최고 87.2<br>(Opus 5와 오차 범위) |
| 단가 (입력 / 출력) | $2 / $10 | $4 / $20 | $10 / $50 |

공식 벤치마크는 medium 이상에서 대부분 Opus 5.5 편이다. 단가는 Fable의 40%다. 무인 실행 문제는 두 모델 모두 지시문으로 다뤄야 한다. 그래서 메인은 Opus 5.5가 기본이다. Fable 5.1은 공식 가이드대로 Opus로 안 풀릴 때 올리는 자리이고, Anthropic 스스로 "실제 격차는 점수보다 좁다"고 한 만큼 그 자리에서 제값을 할 수 있다.

그리고 "혼자 계속 가는가"는 모델만의 문제가 아니다. Claude Code에는 `/goal`이 있다.
완료 조건을 한 줄로 적어 두면, 턴이 끝날 때마다 작은 모델(기본 Haiku)이 조건이 충족됐는지 판정하고, 아니면 사람 대신 다음 턴을 시작한다.

```console
> /goal test/auth의 테스트가 전부 통과하고 lint가 깨끗하다
```

Opus 5.5가 "여기까지 했습니다, 계속할까요?"로 턴을 끝내도, 평가기가 "아직 아님"이라고 답하면 그대로 다음 턴이 돈다. 즉 **목표를 한 줄로 적을 수 있는 작업이면 멈추는 버릇의 대부분을 `/goal`이 메워 준다.** 다만 `/goal`은 권한 모드를 바꾸지 않아서, 무인으로 돌리려면 auto 모드와 같이 써야 한다. 도구 호출 없이 몇 턴 연속 답만 하면 `/goal`도 루프를 멈추고 사람에게 돌려준다.[^goal] Opus 5.5 가이드가 권하는 "완료 조건을 작은 모델로 판정하는 방식"을 Claude Code가 기능으로 넣은 셈이다. 평가 비용은 Haiku라 미미하고, 서브에이전트가 돌고 있으면 평가를 미뤘다가 끝나면 이어간다.

그래서 메인은 일의 모양에 따라 이렇게 고른다.

- **목표가 뚜렷한 장기 작업은 `/goal` + Opus 5.5 `high`.** "모든 테스트 통과", "설계 문서의 수용 기준 전부 충족", "이슈 큐 비우기"처럼 끝을 검증할 수 있는 일이다. Superpowers로 플랜을 실행하는 것도 여기에 든다. 플랜이 곧 완료 조건이다.
- **끝을 한 줄로 못 적는 탐색적인 장기 작업은 Opus 5.5 `high` + 무인 실행 지시문으로 시작한다.** 가다가 판단을 바꿔야 하는 일이다. 평가기가 판정할 조건이 없으니 모델 스스로 계속 가야 하고, 그래서 가이드의 지시문을 CLAUDE.md에 넣는다. `xhigh`로 올려도 막히면 **Fable 5.1 `high`**로 올린다. 이때도 Fable 5.1 가이드의 지시문을 넣고, 범위 밖 수정은 `tier-max` 최종 리뷰에서 거른다. 공식 가이드의 순서 그대로다.
- **페어 코딩하듯 같이 볼 때는 Opus 5.5 `high`.** 터미널 작업과 장기 판단 벤치마크가 좋고, 단가는 Fable의 40%다. 자꾸 멈춰서 묻는 건 옆에 사람이 있으면 단점이 아니라 확인 기회다. Fable 5.1 가이드도 사용자에게 "계속"을 묻는 건 "suits pair programming and other human-in-the-loop work"라고 적었다. 개인적으로는 장기 작업만 아니면 Opus가 페어 코딩하듯 쓰기에 가장 좋다고 본다. 한 단계 하고 보여주고, 방향을 확인하고, 다음 단계로 가는 리듬이 사람과 맞는다. Opus로 안 풀리는 문제만 Fable로 넘긴다.
- **플랜이 이미 있고 Native로 돌린다면 Sonnet 5.5 `high`.** 판단은 플랜이 이미 했으니, 잘 정의된 일상 작업에 맞춘 모델로 충분하다. Anthropic도 Sonnet 5.5를 "well-scoped everyday tasks"에 가장 강하다고 소개한다.[^sonnet-page] Superpowers도 Native 실행은 중간 티어 세션 모델로 잘 돈다고 적었다. 다만 TB 4.0에서 Sonnet `high`(43.0%)는 Opus `medium`(57.6%)보다 낮으니, 막히는 태스크가 잦으면 Opus `medium`으로 바꾼다.

> 이 글의 비용 예시 1~3은 메인을 Opus 5.5 `xhigh`로 둔 경우이고, 예 4는 `high`와 `xhigh`를 나눠 계산했습니다. 권장하는 메인은 Opus 5.5 `high`입니다. 메인을 Fable로 바꿔도 서브에이전트 프리셋과 배선은 그대로 쓸 수 있고, 서브에이전트 비용은 같습니다. 메인 자리의 토큰 단가만 2.5배가 됩니다.
{: .prompt-info }

## 서브에이전트 프리셋 7종 {#presets}

> Superpowers 티어 4종 + 일상 작업 3종입니다. description에 용도만 적고 본문은 비웁니다.
{: .prompt-tip }

### 설계 원칙

- **Superpowers가 쓰는 말과 1:1로 맞춘다.** 스킬은 cheap / standard / most capable 같은 티어로만 말한다. 프리셋 이름을 티어로 지으면 배선 표가 그대로 이름이 되고, 모델이 바뀌어도 `model` 줄만 갈아끼우면 된다.
- **Superpowers에 없는 일은 용도로 이름을 짓는다.** 검색과 요약, 코드 탐색은 Superpowers 역할이 아니다. 메인이 description만 보고 고를 수 있게 용도를 이름으로 둔다.
- **본문(시스템 프롬프트)은 비운다.** 본문을 적으면 서브에이전트가 그 역할로 제한된다. 공식 문서 기준으로 본문은 필수가 아니다. 다만 빈 본문은 Claude Code v2.1.281 이상에서 지원한다.
- **모델은 별칭으로 적는다.** `opus`, `sonnet`, `haiku`는 그 시점의 최신 모델로 풀리니 새 모델이 나와도 프리셋을 안 고쳐도 된다. 공식 문서에 따르면 메인이 같은 계열이면 메인과 같은 모델로 풀려서, 메인이 `opus[1m]`이면 서브에이전트도 1M 컨텍스트를 받는다.

### 근거가 된 코딩 · 설계 벤치마크 {#coding-design-benchmarks}

프리셋 모델을 고를 때 본 벤치마크다. 수치는 2026-10-01에 원출처에서 다시 확인했다.

**같은 기준으로 잰 점수.** Opus 5.5 시스템 카드의 요약 표 하나에서 가져왔다.[^opus-sc] 벤치마크 버전이 모두 같다. Opus 5.5는 max effort(Terminal-Bench 4.0만 xhigh), 비교 모델은 max effort 값이다. Sonnet 5.5는 이 표에 없다. Fable 5.1의 HLE 도구 사용 점수는 Fable 시스템 카드에는 65.0%로 적혀 있다.

| 벤치마크 | Opus 5.5 | Fable 5.1 | Opus 5 (이전 세대) |
|:--|--:|--:|--:|
| SWE-bench Pro | **89.9%** | 81.2% | 79.2% |
| SWE-bench Multilingual | **93.9%** | 89.1% | 89.5% |
| SWE-bench Multimodal | **61.4%** | 54.7% | 59.4% |
| FrontierCode v1.1 (Main) | **54.4%** | 50.3% | 48.0% |
| Terminal-Bench 4.0 | **66.4%** | 55.8% | 52.3% |
| Humanity's Last Exam (도구 없음 / 도구) | **64.4% / 67.7%** | 60.9% / 65.6% | 56.6% / 63.6% |
| GDPval-AA v2.1 (Elo) | **1846** | 1735 | 1708 |
| 입력 / 출력 단가 | $4 / $20 | $10 / $50 | $5 / $25 |

- **SWE-bench Pro**: 실제로 유지보수 중인 저장소에서 여러 파일에 걸친 큰 diff를 요구하는 문제다. 출시 페이지에는 없고 시스템 카드에만 있다.
- **FrontierCode**: 코드 변경이 사람 손을 안 거치고 머지될 수 있는지를 본다. 테스트와 함께 코드 품질 루브릭으로 채점하고, 요청 범위 밖의 수정은 좋은 수정이어도 감점한다. 리뷰어 · 설계 자리에 가장 가깝다.

**같은 effort끼리 비교.** 출시 페이지 차트 원자료에서 옮겼다.[^opus-page][^sonnet-page] 프리셋에 쓰는 effort(medium ~ xhigh)가 핵심이다. FrontierCode는 Claude Code로, CursorBench는 Cursor 자체 에이전트로 쟀다.

| 벤치마크 | 모델 | low | medium | high | xhigh | max |
|:--|:--|--:|--:|--:|--:|--:|
| FrontierCode 1.1 | Sonnet 5.5 | 29.3% | 36.5% | 49.4% | **52.1%** | 46.2% |
| | Opus 5.5 | 47.3% | **54.6%** | **54.0%** | 51.4% | **54.4%** |
| | Fable 5.1 | **52.8%** | 50.9% | 50.3% | 48.7% | 50.3% |
| CursorBench 4.0 | Sonnet 5.5 | 35.8% | 39.2% | 47.8% | 53.1% | 55.5% |
| | Opus 5.5 | 43.7% | **52.5%** | **56.0%** | **56.0%** | **57.8%** |
| | Fable 5.1 | **45.1%** | 46.8% | 49.2% | 51.6% | 51.8% |

Terminal-Bench 4.0의 effort별 표는 [메인 에이전트 절](#choosing-main)에 있다.

읽을 점은 셋이다.

- **medium · high에서는 두 벤치마크 모두 Opus 5.5가 가장 높다.** xhigh에서도 CursorBench는 Opus 5.5가 앞서지만, FrontierCode는 Sonnet 5.5(52.1%)가 Opus 5.5(51.4%)보다 조금 높다. Opus 5.5의 FrontierCode 최고점은 `medium`(54.6%)이다.
- **Fable 5.1은 low에서 가장 높다.** effort 이름이 모델마다 같은 양의 생각을 뜻하지 않아서다. Fable 5.1 가이드도 "작은 모델을 높은 effort로 돌릴 자리라면 Fable 5.1 low도 비교에 넣어 보라"고 권한다.[^fable-prompting]
- **FrontierCode에서 Fable 5.1이 effort를 올릴수록 떨어지는 건 채점 방식 탓이 크다.** CursorBench와 Terminal-Bench에서는 Fable도 effort를 올릴수록 오른다. 위 표(Cognition 측정)에서는 low가 최고점이고, Fable 시스템 카드의 자체 측정에서는 medium이 최고점이다. 시스템 카드에 따르면 Fable 5.1은 effort가 높을수록 요청 밖 파일에 작은 수정(옆 파일의 문서 주석, 새 CI 작업 등)을 더 하고, FrontierCode는 이를 실패로 친다.[^fable-sc] 못 해서가 아니라 더 해서 깎인 점수다. Opus 5.5도 FrontierCode에서 high · xhigh로 가면 같은 이유로 점수가 떨어지고, max에서 거의 회복한다.[^opus-sc]
- **DeepSWE는 이유가 조금 다르다.** Fable 5.1이 모호한 과제를 요구보다 꼼꼼하게 구현했다가(입력 검증, 코드베이스 관례 따르기), 참조 해법 하나에 맞춰 쓴 숨은 테스트에서 떨어졌다고 시스템 카드는 설명한다.[^fable-sc]

그래서 구현 쪽 `tier-cheap` · `tier-standard`는 Sonnet 5.5, 판단 쪽 `tier-capable` · `tier-max`는 Opus 5.5로 나눴다.

- **Sonnet 5.5**: 프리셋이 쓰는 medium · high에서는 같은 effort의 Opus보다 낮다(xhigh 이상에서는 FrontierCode xhigh, Terminal-Bench max처럼 Opus를 넘기도 한다). 대신 과제당 비용이 훨씬 싸다. 예를 들어 FrontierCode에서 Sonnet `high`는 49.4%에 $0.42, Opus `medium`은 54.6%에 $0.80이다. 시그니처와 테스트가 정해진 태스크는 틀려도 테스트와 리뷰가 잡는다. Anthropic도 Sonnet 5.5를 "well-scoped everyday tasks"에 가장 강하다고 두고, "Opus 5.5 remains clearly stronger at complex, open-ended work requiring sustained judgment"라고 적었다.[^sonnet-page]
- **Opus 5.5**: 프리셋에 쓰는 medium ~ xhigh에서 FrontierCode xhigh 하나를 빼면 세 벤치마크 모두 가장 높고, 공식 문서도 대부분 작업의 시작점으로 둔다. Claude Code로 잰 Terminal-Bench와 FrontierCode에서도 같다.
- **Fable 5.1**: 프리셋 effort에서는 Opus 5.5보다 낮은데 단가는 2.5배라 기본 프리셋에서 뺐다. 그렇다고 못한 모델은 아니다. 공식 가이드가 Opus 다음 단계로 두는 모델이라, [왜 Fable은 없나](#why-no-fable)에서 사람이 직접 부르는 자리로 남겨 둔다.

**설계 · 아키텍처.** 5.x 모델의 설계 능력을 직접 재는 공개 벤치마크는 찾지 못했다. 대신 두 모델의 시스템 카드가 설계와 판단의 약점을 직접 적어 두었다.

- **Opus 5.5** (시스템 카드): 내부 사용에서 리뷰 피드백을 좁게만 반영하고 "without reconsidering whether the overall design is right", 즉 전체 설계가 맞는지는 다시 보지 않았다. 플랜을 그 플랜이 도와야 할 사람이 아니라 자기가 쓴 요구사항에 대고 검증하기도 했다.[^opus-sc]
- **Opus 5.5 vs Fable 5.1** (같은 시스템 카드의 METR 평가): 연구자의 판단력이나 "taste" 같은 능력에서 Opus 5.5가 Fable 5.1보다 크게 나아졌다는 근거가 없다.[^opus-sc]
- **Fable 5.1 / Mythos 5.1** (시스템 카드, 생물 분야 레드팀): 새 아이디어가 드물고, 어떻게 프롬프트해도 같은 실험 설계로 수렴하는 경향이 있었다. 사용자가 준 틀을 그대로 이어 가고, 계획을 지나치게 낙관적으로 내놓는다.[^fable-sc] 소프트웨어 설계가 아니라 생물 연구 설계에서 나온 관찰이다.

두 시스템 카드 모두 열린 문제에서의 판단을 약점으로 적었고, METR는 연구 판단력("taste")에서 Opus 5.5가 Fable 5.1보다 크게 나아졌다는 근거가 없다고 봤다. 설계 산출물은 모델 등급과 상관없이 사람이 다시 봐야 한다.

학계에는 아키텍처 · 설계를 따로 재는 벤치마크도 있다. 다만 <mark>공개된 Claude 점수는 4.x 이전 모델까지이고, 5.x 점수는 아직 없다.</mark>

| 벤치마크 | 재는 것 | Claude 결과 | 출처 |
|:--|:--|:--|:--|
| R2ABench | 요구사항 문서(PRD)를 읽고 아키텍처 다이어그램 작성 | Sonnet 4.6: 구성요소 F1 0.54,<br>구성요소 사이 관계 F1 0.09 | arXiv 2604.06683 v1 (2026-04) |
| SWE-Lancer Manager | 실제 프리랜서 과제에서 경쟁하는 구현안 중 최선을 고르기 | Claude 3.5 Sonnet 44.9%<br>(Diamond 세트 1위, o1보다 3.4pt 높음) | arXiv 2502.12115 v4 (2025-05) |
| SAKE | 소프트웨어 아키텍처 지식 객관식 2,154문항 | Haiku 4.5 92.2%, Sonnet 4.6 93.7%,<br>Opus 4.6 93.6% (zero-shot) | arXiv 2606.29520 (2026-06) |
| SADU | 아키텍처 다이어그램 읽고 이해하기 | Sonnet 4.5 56.4% (1위 Gemini 3 Flash 70.2%) | arXiv 2604.04009 (2026-04) |

모델 세대가 달라서 프리셋을 고르는 근거로 직접 쓰지는 않았다. 그래도 시스템 카드의 서술과 같은 방향을 가리킨다. SAKE처럼 아키텍처 **지식**을 묻는 시험은 Haiku부터 Opus까지 93% 안팎으로 차이가 거의 없다. 반대로 R2ABench에서 직접 **설계**를 시키면, 구성요소는 절반 정도 맞히지만 구성요소 사이의 관계는 거의 틀린다. 설계 결과물은 관계와 경계를 사람이 다시 봐야 한다.

### A. Superpowers 티어 4종

비용과 첫 응답 시간은 앞의 Artificial Analysis 표에서 가져왔다. (같은 평가 세트 한 벌 기준 출력 비용)

| 프리셋 | 모델 · effort | Superpowers에서 부르는 말 | 지수 | 출력 비용 | 첫 응답 |
|:--|:--|:--|--:|--:|--:|
| `tier-cheap` | Sonnet 5.5 medium | cheap / fast model | 41 | $290 | 1.3초 |
| `tier-standard` | Sonnet 5.5 high | standard / mid-tier | 47 | $500 | 13초 |
| `tier-capable` | Opus 5.5 high | one tier above (에스컬레이션) | 54 | $1,060 | 39초 |
| `tier-max` | Opus 5.5 xhigh | most capable | 56 | $2,000 | 134초 |

**`tier-cheap`** — Anthropic 출시 페이지 기준으로 Sonnet 5.5는 medium에서 Terminal-Bench 4.0 28.8%로 Sonnet 5의 최고 점수(10.3%, max)를 넘기면서 비용은 1/10 이하였다.[^sonnet-page] 시그니처와 테스트가 정해진 구현은 틀려도 테스트가 잡아 주니 여기까지로 충분하다.

```markdown
---
name: tier-cheap
description: 시그니처와 테스트가 이미 정해진 구현, 단일 파일 수정, 작은 diff 재리뷰. Superpowers의 cheap / fast model 티어
model: sonnet
effort: medium
---
```
{: file=".claude/agents/tier-cheap.md" }

**`tier-standard`** — Anthropic 출시 페이지에 따르면 Sonnet 5.5는 FrontierCode에서 high로 GPT-6 Sol의 최고 점수(49.3%)와 같은 점수를 비용 약 1/5로 낸다. 표의 46.2%는 max 값이고, max가 xhigh보다 낮다. 여러 파일을 오가며 통합하거나 디버깅할 때, 그리고 태스크 하나의 diff를 스펙과 대조하는 리뷰에 쓴다. Superpowers가 "리뷰어의 하한은 중간 티어"라고 한 자리다.

```markdown
---
name: tier-standard
description: 여러 파일에 걸친 통합 구현, 디버깅, 태스크 단위 코드 리뷰. Superpowers의 standard / mid-tier 티어
model: sonnet
effort: high
---
```
{: file=".claude/agents/tier-standard.md" }

**`tier-capable`** — Terminal-Bench 4.0에서 Opus 5.5 high는 64.2%($3.88), xhigh는 66.4%($7.35)다. 점수의 97%를 절반 비용으로 얻는다. Superpowers가 "막힌 구현자보다 한 티어 위"라고 한 에스컬레이션 자리와, 모델을 지정하지 않는 단독 코드 리뷰 템플릿에 쓴다.

```markdown
---
name: tier-capable
description: 두 번 이상 막힌 구현의 에스컬레이션, 일반 코드 리뷰. Superpowers의 one tier above 티어
model: opus
effort: high
---
```
{: file=".claude/agents/tier-capable.md" }

**`tier-max`** — Opus 5.5의 Terminal-Bench 4.0 최고점(66.4%)이 xhigh에서 나온다. max는 64.8%로 xhigh와 오차 범위 안에서 차이가 없는데 비용은 1.5배라 넣지 않았다. 설계 판단이 필요한 구현, 동시성이나 보안처럼 위험한 diff의 리뷰, 브랜치 전체를 보는 최종 리뷰처럼 한 번 틀리면 되돌리기 비싼 일에만 쓴다.

설계 · 아키텍처를 재는 학계 벤치마크는 있지만 5.x 모델 점수는 아직 없다. 대신 시스템 카드의 같은 표에서 Opus 5.5는 다학제 추론(Humanity's Last Exam, 도구 없음 64.4% vs 60.9%)과 지식 노동 평가(GDPval-AA v2.1, 1846 vs 1735)에서 Fable 5.1보다 높다.[^opus-sc] 다만 시스템 카드는 Opus 5.5가 리뷰 피드백을 좁게 반영하고 전체 설계를 다시 보지 않는 약점도 적었다. 그래서 `tier-max`에 설계 리뷰를 맡길 때는 "diff만 보지 말고 전체 설계가 맞는지부터 다시 보라"고 호출 지시에 적어 주는 게 좋다. **`tier-max`로도 답이 안 나오는 문제는 `model: fable`로 따로 부른다.** 공식 가이드가 권하는 에스컬레이션 순서다. 아래 [왜 Fable은 없나](#why-no-fable)에서 다시 다룬다.

```markdown
---
name: tier-max
description: 설계 판단이 필요한 구현, 동시성 · 보안 · 데이터 손실처럼 위험한 변경의 리뷰, 머지 전 브랜치 전체 최종 리뷰. Superpowers의 most capable 티어
model: opus
effort: xhigh
---
```
{: file=".claude/agents/tier-max.md" }

### B. 일상 작업 3종

| 프리셋 | 모델 · effort | 맡는 일 | 근거 |
|:--|:--|:--|:--|
| `search` | Haiku 4.5 | 검색 한 번으로 끝나는 확인,<br>문서 · 로그 한 건 요약처럼 한 번에 끝나는 일 | 단가가 Sonnet의 절반,<br>첫 응답 0.7초, 생각 토큰 없음 |
| `web-research` | Sonnet 5.5 medium | 여러 페이지를 읽고 대조하는 웹 조사,<br>버전 · 플래그 · 벤치마크 수치 확인 | Opus 5.5 low와 지수가 비슷한데($290 vs $400)<br>첫 응답이 10배 빠름 |
| `explore` | Sonnet 5.5 low | 여러 파일을 뒤지는 코드 조사,<br>호출 관계 추적 | 여러 턴 작업에서 Haiku(reasoning)보다<br>싸고($230 vs $390) 지수 2배 |

**`search`** — 한 번 읽고 한 번 답하는 일은 생각 토큰이 필요 없다. Haiku 4.5는 effort를 지원하지 않지만 이런 일에는 조절할 것도 없다. 공식 문서도 단순한 서브에이전트 작업에는 `model: haiku`를 권한다.

```markdown
---
name: search
description: 검색 한 번으로 끝나는 확인, 문서나 로그 한 건 요약처럼 한 번 읽고 한 번 답하면 끝나는 조사. 여러 웹 페이지를 읽고 대조해야 하면 web-research, 여러 파일을 뒤져야 하면 explore를 쓴다
model: haiku
---
```
{: file=".claude/agents/search.md" }

**`web-research`** — 웹 조사는 한 번에 끝나지 않는다. 검색하고, 페이지를 몇 개 열고, 출처끼리 수치가 다르면 다시 찾는다. 턴이 쌓이는 일이라 `explore`와 같은 이유로 Haiku가 아니라 Sonnet 5.5다. effort는 low가 아니라 medium으로 올렸다. 어느 출처가 1차 자료인지, 수치가 어느 effort에서 나온 건지 가려야 하기 때문이다. Sonnet 5.5에서 low → medium은 1점당 $12로 가장 싼 구간이다. Opus 5.5 low(지수 42, $400, 첫 응답 13.5초)와 비교하면 Sonnet 5.5 medium(지수 41, $290, 1.3초)이 점수는 비슷하고 더 싸고 빠르다.

```markdown
---
name: web-research
description: 여러 웹 페이지를 읽고 대조하는 조사, 공식 문서 · 릴리스 노트 · 리더보드에서 버전 · 플래그 · 수치 확인. 출처 URL과 페이지 날짜를 같이 돌려준다
model: sonnet
effort: medium
---
```
{: file=".claude/agents/web-research.md" }

**`explore`** — 파일을 여러 개 열어 보며 답을 찾는 조사는 턴이 쌓인다. Artificial Analysis 측정에서 Haiku 4.5(reasoning)는 지수 17에 출력 토큰 78M($390), Sonnet 5.5 low는 지수 36에 23M($230)이다. Superpowers가 경고한 "싼 모델이 턴을 2~3배 쓴다"가 그대로 나타난다. 이름을 `Explore`로 두면 기본 Explore 서브에이전트를 덮어쓸 수도 있지만, 기본 동작을 바꾸는 일이라 별도 이름으로 시작하는 편이 안전하다.

```markdown
---
name: explore
description: 여러 파일을 뒤지는 코드 조사, 호출 관계와 데이터 흐름 추적, 변경 영향 범위 파악
model: sonnet
effort: low
---
```
{: file=".claude/agents/explore.md" }

<!-- TODO: Haiku 4.5 수치(78M 토큰)는 Artificial Analysis의 다른 지수 버전일 수 있음. 발표 전 확인 -->

### 별칭 대신 전체 ID로 고정해야 할 때

두 경우에는 `model: claude-opus-5-5`처럼 전체 ID로 바꾼다.

- **제공자가 Anthropic API가 아닐 때.** 별칭은 제공자마다 다르게 풀린다. `opus`가 Microsoft Foundry에서는 Opus 4.6, `sonnet`이 Amazon Bedrock에서는 Sonnet 4.5다.
- **모델 × effort 조합을 고정하고 싶을 때.** 새 버전이 나오면 같은 effort라도 토큰 사용량과 기본값이 달라지고, 지원하지 않는 effort는 경고 없이 한 단계 아래로 떨어진다. `xhigh`가 Opus 4.6에서는 `high`로 실행된다. 이 글의 벤치마크 근거는 Sonnet 5.5와 Opus 5.5 기준이라, 그 근거를 그대로 지키려면 고정해야 한다.

### 왜 Fable은 없나 {#why-no-fable}

Fable 5.1이 못해서 뺀 게 아니다. 공식 문서는 Fable 5.1을 일반 고객이 쓸 수 있는 가장 강한 모델로 둔다.[^choosing] 뺀 이유는 프리셋이 쓰는 effort 구간의 비용 대비 점수다.

- 프리셋이 쓰는 medium ~ xhigh에서는 코딩 벤치마크 점수가 Opus 5.5보다 낮다.
- 출력 단가는 Opus의 2.5배다. xhigh 기준 지수 53에 $6,000이라 `tier-max`(지수 56, $2,000)에 밀린다.

그래도 Fable을 부를 자리는 남겨 둔다. 공식 모델 선택 가이드는 "Opus 5.5를 xhigh · max로 올려도 어려운 추론이나 장기 에이전트 작업이 모자라면 Fable 5.1로 옮기라"고 한다.[^choosing]
`tier-max`로도 답이 안 나오는 설계 문제나 어려운 디버깅은 사람이 직접 `tier-max`를 부르면서 `model: fable`을 넘긴다. effort는 프리셋의 `xhigh`가 그대로 적용된다. 프리셋을 부를 때 model을 넘기지 않는다는 배선 규칙의 유일한 예외다.
Superpowers가 "막힌 구현자보다 한 티어 위"라고 한 에스컬레이션의 마지막 단이 이 자리다.
Fable은 요청보다 더 고치는 경향이 있으니, 부를 때 가이드의 범위 지시문("요청 밖의 버그나 개선은 고치지 말고 요약에 후속 과제로 적어라")을 함께 넘기면 좋다.[^fable-prompting]
정해진 프리셋으로 만들지 않은 이유는, 자동 위임이 Fable로 새는 것을 막기 위해서다. 사람이 판단해서 부르는 게 맞다.

### 프리셋을 두는 위치

| 위치 | 범위 | 우선순위 |
|:--|:--|--:|
| managed settings | 조직 전체 | 1 |
| `--agents` CLI 플래그 | 현재 세션 | 2 |
| `.claude/agents/` | 현재 프로젝트 (git으로 팀 공유) | 3 |
| `~/.claude/agents/` | 내 모든 프로젝트 | 4 |
| 플러그인의 `agents/` | 플러그인을 켠 곳 | 5 |

같은 이름이 여러 곳에 있으면 우선순위가 높은 쪽이 쓰인다.

## CLAUDE.md에 추가하기 {#claude-md}

> description만으로는 "부를 수 있다"이지 "부른다"가 아닙니다. 실제로 쓰이게 하려면 CLAUDE.md 라우팅이나 이름 지목이 필요합니다.
{: .prompt-tip }

description만 적은 프리셋을 CLAUDE.md에 언급하지 않아도 호출될까? 문서상으로는 된다.[^auto-delegation]

> Claude automatically delegates tasks based on the task description in your request, the `description` field in subagent configurations, and current context.

그런데 실제로 써 보면 프리셋을 만들어 두기만 해서는 잘 안 불린다. 이건 description을 잘못 써서가 아니다. 근거가 둘 있다.

1. **문서도 "보통은"이라고만 한다.** 이름을 지목해도 "Claude typically delegates"라고 적혀 있다. 보장이 아니다.
2. **기본 서브에이전트가 경쟁한다.** 읽기 전용 탐색은 기본 Explore가, 여러 단계 작업은 general-purpose가 먼저 후보에 오른다. 내가 만든 `explore`가 있어도 메인은 익숙한 Explore를 고를 수 있다.

그래서 프리셋이 실제로 쓰이게 하려면 세 가지 중 하나가 더 필요하다.

| 방법 | 어떻게 | 언제 |
|:--|:--|:--|
| CLAUDE.md 라우팅 | "이런 일은 이 프리셋"을 적는다 | 매 세션 자동으로 적용하고 싶을 때 (이 글의 방식) |
| 이름 지목 | 프롬프트에 "tier-max로 리뷰해"<br>또는 `@agent-tier-max` | 한 번만 확실히 부르고 싶을 때.<br>`@` 지목은 선택을 메인에게 맡기지 않는다 |
| `--agent` | `claude --agent tier-standard` | 세션 전체를 그 프리셋 설정으로 돌릴 때 |

description은 그래도 잘 써야 한다. 문서가 권하는 건 "언제 쓰는지"를 적고 "use proactively" 같은 문구를 넣는 것이다. 이 글의 프리셋은 용도를 한 줄로 적고 Superpowers 티어 이름을 붙여서, 배선 표의 단어와 description의 단어가 같게 했다. 메인이 표를 읽고 프리셋을 찾을 때 헷갈리지 않게 하기 위해서다.

CLAUDE.md 조각은 두 판이다. Superpowers를 안 쓰면 **기본판**, 쓰면 다음 절의 **배선판**을 넣는다.

### 기본판 — Superpowers 없이 쓸 때

일곱 프리셋 모두를 상황으로 라우팅한다. 기본 Explore 대신 `explore`를 쓰라는 것도 여기서 정한다.

```markdown
## 서브에이전트 라우팅

- 검색 한 번으로 끝나는 확인, 문서 · 로그 한 건 요약: search
- 여러 웹 페이지를 읽고 대조하는 조사, 버전 · 플래그 · 수치 확인: web-research
- 여러 파일을 뒤지는 코드 조사, 영향 범위 파악: explore (기본 Explore 대신 쓴다)
- 시그니처와 테스트가 정해진 구현, 단일 파일 수정, 작은 diff 재리뷰: tier-cheap
- 여러 파일에 걸친 구현, 디버깅, 태스크 단위 코드 리뷰: tier-standard
- 두 번 이상 막힌 구현의 에스컬레이션, 일반 코드 리뷰: tier-capable
- 설계 판단이 필요한 구현, 동시성 · 보안 등 위험한 변경의 리뷰, 머지 전 최종 리뷰: tier-max
- 위 프리셋을 부를 때 model 값은 넘기지 않는다. 프리셋의 모델과 effort를 그대로 쓴다.
```
{: file="CLAUDE.md" }

<details markdown="1">
<summary>English version (basic)</summary>

```markdown
## Subagent routing

- One-shot lookups, one-shot summaries of a single doc or log: search
- Multi-page web research, checking versions / flags / figures against sources: web-research
- Multi-file code investigation, impact analysis: explore (use instead of the built-in Explore)
- Implementation with signatures and tests already fixed, single-file fixes, scoped re-review of a small diff: tier-cheap
- Multi-file implementation, debugging, task-level code review: tier-standard
- Escalation after two or more failed attempts, general code review: tier-capable
- Design-judgment implementation, review of risky changes (concurrency, security), final review before merge: tier-max
- When dispatching these presets, do NOT pass a model value. Use the preset's model and effort as is.
```
{: file="CLAUDE.md" }

</details>

### 배선판 — Superpowers를 쓸 때

Superpowers 템플릿은 `general-purpose`를 이름으로 부르고 `model`을 직접 넘긴다. 이걸 `tier-*`로 바꿔 부르라는 지시가 필요하다. 다음 절에서 다룬다.

CLAUDE.md는 매 세션 컨텍스트에 올라간다. 공식 문서도 200줄 이하를 권장하니, 라우팅처럼 짧은 규칙만 두고 긴 절차는 스킬로 뺀다.

## Superpowers를 쓴다면: 배선 만들기 {#wiring}

> Superpowers의 모델 티어를 `tier-*` 프리셋에 1:1로 연결해야 모델과 effort가 함께 고정됩니다.
{: .prompt-tip }

앞에서 본 세 문제를 막는 게 배선이다. 먼저 서브에이전트의 모델이 정해지는 순서를 알아야 한다.

1. 호출할 때 넘긴 `model` 값
2. 프리셋 frontmatter의 `model`
3. `CLAUDE_CODE_SUBAGENT_MODEL` 환경변수
4. 메인 모델

Superpowers는 호출할 때 `model`을 넘기도록 되어 있고, 이 값은 프리셋의 `model`보다 우선한다.
그래서 배선 규칙은 두 가지다.

- Superpowers가 `general-purpose`를 부르라고 할 때, **해당 티어의 `tier-*` 프리셋을 대신 쓴다.**
- 프리셋을 쓸 때는 **`model` 값을 넘기지 않는다.** 프리셋에 적힌 모델과 effort가 그대로 쓰이게 하기 위해서다.

Superpowers 스킬 원문의 표현을 왼쪽에, 프리셋을 오른쪽에 두면 배선 표는 이렇게 된다.

```markdown
## Superpowers 배선

Superpowers 스킬이 `Subagent (general-purpose)` 템플릿으로 서브에이전트를 부르라고 하면,
general-purpose 대신 아래 프리셋을 subagent_type으로 쓰고 model 값은 넘기지 않는다.
템플릿의 `model:` 줄은 이 표로 대신한다.

| Superpowers 원문 | 상황 | 프리셋 |
|---|---|---|
| cheapest tier / fast, cheap model | 플랜에 코드 본문이 있는 구현, 1~2파일 독립 함수, 단일 파일 수정, 작은 diff 재리뷰 | tier-cheap |
| standard model / mid-tier floor | 설명 위주 플랜 구현, 여러 파일 통합, 디버깅, 태스크 리뷰 | tier-standard |
| a model at least one tier above | 수정 4~5회차에 막힌 구현자 교체 (cheap→standard, standard→capable) | tier-capable |
| (model 줄 없음) code-reviewer.md | requesting-code-review 단독 호출 | tier-capable |
| most capable available model | 설계 판단 구현, 동시성 · 보안 등 위험한 diff 리뷰, 최종 브랜치 리뷰 | tier-max |
| orchestrator subagent (mid-tier) | claude-code-tools.md의 오케스트레이터 위임을 쓸 때 | tier-standard |
```
{: file="CLAUDE.md" }

기본판의 `search` · `web-research` · `explore` 세 줄은 그대로 두고, 구현 · 리뷰 네 줄 대신 이 표를 넣는다.
영어 프로젝트라면 아래 판을 쓴다. Superpowers 원문 열은 스킬의 표현 그대로라 두 판이 같다.

<details markdown="1">
<summary>English version (Superpowers wiring)</summary>

```markdown
## Subagent routing

- One-shot lookups, one-shot summaries of a single doc or log: search
- Multi-page web research, checking versions / flags / figures against sources: web-research
- Multi-file code investigation, impact analysis: explore (use instead of the built-in Explore)
- Implementation and review follow the Superpowers wiring table below

## Superpowers wiring

When a Superpowers skill tells you to dispatch a `Subagent (general-purpose)` template,
use the preset below as subagent_type instead of general-purpose and do NOT pass a model value.
This table replaces the template's `model:` line.

| Superpowers wording | Situation | Preset |
|---|---|---|
| cheapest tier / fast, cheap model | plan carries the code body; 1-2 file isolated function; single-file fix; scoped re-review of a small diff | tier-cheap |
| standard model / mid-tier floor | prose-only plan; multi-file integration; debugging; task review | tier-standard |
| a model at least one tier above | replace a stuck implementer at fix rounds 4-5 (cheap->standard, standard->capable) | tier-capable |
| (no model line) code-reviewer.md | standalone requesting-code-review dispatch | tier-capable |
| most capable available model | design-judgment implementation; risky diffs (concurrency, security); final whole-branch review | tier-max |
| orchestrator subagent (mid-tier) | when using the claude-code-tools.md orchestrator delegation | tier-standard |
```
{: file="CLAUDE.md" }

</details>

이렇게 두면 "가장 강한 모델"이 매번 Opus 5.5 `xhigh`로 고정되고, 티어마다 모델과 effort가 실행에 상관없이 같게 유지된다.
어느 태스크를 어느 티어로 보낼지 흔들리는 문제는 플랜 리뷰 때 사람이 태스크별로 확인해서 줄인다.

> `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`가 켜져 있으면 프리셋과 호출 값을 모두 무시하고, 모든 서브에이전트가 `CLAUDE_CODE_SUBAGENT_MODEL`의 모델로 실행됩니다. 배선이 안 먹는 것 같으면 이 환경변수부터 확인하세요.
{: .prompt-warning }

Superpowers 쪽에도 Claude Code 전용으로 비용을 줄이는 방법이 하나 있다.
태스크를 조율하는 메인 자리가 가장 비싸니, 사용자가 원하면 **중간 티어 모델의 오케스트레이터 서브에이전트 하나**에게 플랜 전체 실행을 맡기는 방식이다.
메인은 결과와 오케스트레이터가 내린 판단 목록만 받는다. 릴리스 노트 기준으로 비용과 시간이 약 절반으로 줄었다고 한다.
배선 표의 마지막 줄이 이 자리다.

## 플래닝 이후 리뷰하면 좋은 포인트 {#review-after-planning}

> 구현을 시작하기 전, 플랜이 스펙을 빠짐없이 덮는지 사람이 한 번 확인합니다.
{: .prompt-tip }

Superpowers는 플랜을 쓴 뒤 스스로 점검하고, 저장된 플랜을 사람이 승인해야 실행한다.
이때 사람이 보면 좋은 포인트를 v6.4.2의 자체 점검 항목 기준으로 정리했다.

- **스펙 커버리지**: 스펙의 요구사항마다 그걸 구현하는 태스크가 있는가?
- **Review Focus 섹션**: 스펙이 암시하지만 테스트가 안 다루는 입력이나 실패 상황이 적혀 있는가? 비어 있다면 "확인했는데 없음"인지 확인한다.
- **타입 · 이름 일관성**: 앞 태스크에서 정의한 함수 이름과 시그니처를 뒤 태스크가 그대로 쓰는가?
- **스텝이 결정을 담았는가**: 테스트 스텝에 테스트 이름과 단언이, 코드 스텝에 시그니처 · 파일 · 스펙 값이 있는가? 코드 본문이 적혀 있다면 시그니처와 테스트만으로 정해지지 않는 알고리즘인가?
- **분량**: 플랜이 스펙보다 몇 배 길다면 결정이 아니라 코드를 옮겨 적은 것이다.
- **빈 결정이 없는가**: "TBD", "엣지 케이스 처리", "적절한 검증 추가"처럼 아무것도 정하지 않은 줄은 구현자에게 추측을 떠넘긴다.
- **태스크 난이도 → 프리셋**: 각 태스크가 몇 개 파일을 건드리는지 보고 어느 프리셋으로 갈지 예상해 본다. 설계 판단이 필요한 태스크가 `tier-cheap`으로 가면 안 된다.
- **실행 방식**: Superpowers가 이 플랜에 어느 방식을 추천하는지와 그 이유를 확인한다. Native를 고르면 구현 전체가 메인에서 돌기 때문에, 메인의 모델과 effort가 곧 구현 비용이 된다.

## 머지 전에 리뷰하면 좋은 포인트 {#review-before-merge}

> 최종 리뷰어의 판정을 그대로 믿지 말고, 판정의 근거와 "판단하지 않은 것"을 사람이 확인합니다.
{: .prompt-tip }

Superpowers의 최종 리뷰어는 브랜치 전체를 보고 Critical / Important / Minor로 이슈를 나누고, "머지해도 되나?"에 Yes / No / With fixes로 답한다.
사람이 머지 전에 볼 포인트는 이렇다.

- **Critical 0건**: 버그, 보안, 데이터 손실 위험이 남아 있지 않은가?
- **플랜과의 차이**: 플랜과 다르게 구현된 부분이 있다면, 의도한 개선인지 문제인지 확인한다.
- **"Declined to judge" 목록**: 리뷰어는 "스펙 밖이라 판단하지 않은 것"을 따로 적고, 실행 중인 세션이 하나씩 결정한다. 여기에 진짜 문제가 숨어 있는 경우가 많으니 그 결정이 맞는지 확인한다.
- **실행 중 내린 판단(Ruling) 목록**: Superpowers는 실행 중에 사람에게 묻지 않고 스스로 판단한 내용을 기록한다. 각 판단이 틀렸을 때의 비용까지 적혀 있으니 하나씩 읽어 본다.
- **테스트가 실제 동작을 검증하는가**: 목(mock)만 확인하는 테스트는 아닌가?
- **운영 준비**: 스키마가 바뀌었다면 마이그레이션, 하위 호환성은 고려됐는가?
- **최종 리뷰어의 모델**: 최종 리뷰가 정말 `tier-max`로 돌았는지 확인한다. `~/.claude/projects/**/*.jsonl` 트랜스크립트에 서브에이전트마다 실제로 응답한 모델이 기록된다. 배선이 잘못되면 여기서 조용히 바뀐다.

## 한 걸음 더: triad-dispatch {#triad-dispatch}

> 같은 모델 계열의 리뷰어는 구현자와 같은 사각지대를 공유합니다. 다른 계열에게 한 번 더 물어보세요.
{: .prompt-tip }

Superpowers의 리뷰어도 결국 Claude다. 구현자가 합리화해 버린 버그를 같은 계열의 리뷰어가 똑같이 넘어갈 수 있다.

[triad-dispatch](https://github.com/codefoundry-io/triad-dispatch)는 이 문제를 위해 만든 Claude Code 플러그인이다.
Codex, Gemini, Antigravity 같은 다른 모델 계열의 CLI에 질문을 한 번씩 보내고, 독립적인 판정을 받아 온다.

- 스킬: `triad-codex-dispatch`, `triad-gemini-dispatch`, `triad-antigravity-dispatch`, `triad-cross-family-review`
- Superpowers와 독립적으로 동작하지만 같이 쓰기 좋다. 특히 `triad-cross-family-review`는 subagent-driven-development의 마지막 관문으로 쓰기 좋다.

<!-- TODO: 설치 명령과 사용 예시 추가 -->

## 정리 {#summary}

> 서브에이전트는 effort를 상속받기 때문에, 모델 × effort 프리셋으로 고정하고 Superpowers에 배선해야 합니다.
{: .prompt-tip }

- 서브에이전트는 **effort를 메인에서 상속**받는다. 메인이 xhigh면 모든 서브에이전트가 xhigh다.
- effort는 단가가 아니라 **토큰 양**을 바꾼다. 같은 모델에서도 비용과 시간이 몇 배씩 달라진다.
- 서브에이전트의 effort를 고정하는 방법은 **프리셋 frontmatter**뿐이다.
- 메인은 **Opus 5.5 high**가 기본이다. 목표가 뚜렷하면 **`/goal`**을 붙이고, 탐색적인 장기 작업은 무인 실행 지시문을 넣고, Opus로 막히는 문제만 **Fable 5.1**로 올린다. Native 실행은 Sonnet 5.5도 된다.
- 프리셋은 **Superpowers 티어 4종(`tier-*`) + 일상 작업 3종(`search`, `web-research`, `explore`)**. description에 용도만 적고 본문은 비운다. 만들어 두기만 하면 잘 안 불리니 CLAUDE.md 라우팅이나 `@agent-` 지목으로 연결한다.
- Superpowers는 모델 티어만 정하고 effort는 정하지 않는다. **배선**으로 티어와 프리셋을 연결해야 절약이 완성된다.
- 아낄 곳에서 아끼고, <mark>최종 리뷰처럼 써야 할 곳에는 제대로 쓴다.</mark>

## 내려받기 {#presets-download}

프리셋 7종과 CLAUDE.md 조각 네 가지(기본판 · 배선판 × 한국어 · 영어)를 묶은 zip이다. `.claude/agents/`를 프로젝트에 복사하고, 조각을 CLAUDE.md에 붙여 넣으면 된다.

[claude-subagent-presets.zip 내려받기](/assets/img/posts/2026-09-30-claude-subagent-superpowers/claude-subagent-presets.zip)

<details markdown="1">
<summary>발표자 노트 — 10분 배분</summary>

| 절 | 분 | 한 줄 |
|:--|--:|:--|
| 도입 | 0.5 | 느리고 비쌌다. 원인은 서브에이전트의 effort 상속 |
| 메인/서브 개념 + 상속 | 2 | `/model`로 정한 effort를 서브가 그대로 물려받는다 |
| effort별 비용 · 시간 | 1.5 | xhigh → max는 지수 2점에 토큰 2.6배 |
| Superpowers 소개 + 운영법 | 2 | 스펙 → 플랜 → 태스크별 구현 · 리뷰. 컨텍스트가 안 쌓인다 |
| 가이드대로 하면 + 비용 예시 | 1.5 | 모델은 낮춰도 effort는 xhigh. 배선하면 절반 |
| 메인 선택 | 1 | Opus가 기본, 목표가 뚜렷하면 `/goal`, 막히면 Fable로 올린다 |
| 프리셋 7종 + 배선 | 1 | tier 4종 + search · web-research · explore. 만들기만 하면 안 불린다 |
| 리뷰 포인트 + triad | 0.5 | Declined to judge 목록, 다른 계열 교차 리뷰 |

</details>

## 참고 자료 {#references}

- Claude Code 공식 문서
  - [Subagents](https://code.claude.com/docs/en/sub-agents)
  - [Model configuration (effort, 모델 별칭)](https://code.claude.com/docs/en/model-config)
  - [Manage costs effectively (요청마다 대화 전체 전송)](https://code.claude.com/docs/en/costs)
  - [Keep Claude working toward a goal (/goal)](https://code.claude.com/docs/en/goal)
- Claude Platform 공식 문서
  - [Effort](https://platform.claude.com/docs/en/build-with-claude/effort)
  - [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
  - [Models overview (모델 ID)](https://platform.claude.com/docs/en/about-claude/models/overview)
  - [Prompting Claude Opus 5.5 — 무인 에이전트 실행 절](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5)
  - [Prompting Claude Fable 5.1 — 작업 완료 · 범위 · 서브에이전트 절](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1)
  - [Choosing a model (모델 선택 가이드)](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
  - [What's new in Claude Fable 5.1](https://platform.claude.com/docs/en/models/fable-5-1/whats-new-fable-5-1)
  - [Prompt caching (캐시 단가, 미적중 시 전체 재처리)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Anthropic 출시 페이지
  - [Introducing Claude Opus 5.5](https://www.anthropic.com/claude-opus-5-5)
  - [Introducing Claude Sonnet 5.5](https://www.anthropic.com/claude-sonnet-5-5)
  - [Introducing Claude Fable 5.1 and Claude Mythos 5.1](https://www.anthropic.com/claude-fable-and-mythos-5-1)
  - [Claude Opus 제품 페이지](https://www.anthropic.com/claude/opus)
  - [Opus 5.5 — 긴 코딩 세션 (요청당 컨텍스트 2.6배, 입력:출력 324:1)](https://claude.com/blog/claude-opus-5-5-built-for-coding-sessions-that-use-more-context)
- Anthropic 시스템 카드
  - [Claude Opus 5.5 System Card (2026-09-22)](https://www-cdn.anthropic.com/fc1b44717c85dc068bc6ba5024219938094694bd/Claude%20Opus%205.5%20System%20Card.pdf)
  - [Claude Fable 5.1 & Claude Mythos 5.1 System Card (2026-09-01)](https://www-cdn.anthropic.com/0339e6a7c5c7b87f5c07798616dc32c215d14235/Claude%20Fable%205.1%20&%20Claude%20Mythos%205.1%20System%20Card.pdf)
- Artificial Analysis 측정값
  - Opus 5.5: [low](https://artificialanalysis.ai/models/claude-opus-5-5-low) · [medium](https://artificialanalysis.ai/models/claude-opus-5-5-medium) · [high](https://artificialanalysis.ai/models/claude-opus-5-5-high) · [xhigh](https://artificialanalysis.ai/models/claude-opus-5-5-xhigh) · [max](https://artificialanalysis.ai/models/claude-opus-5-5)
  - Fable 5.1: [low](https://artificialanalysis.ai/models/claude-fable-5-1-low) · [medium](https://artificialanalysis.ai/models/claude-fable-5-1-medium) · [high](https://artificialanalysis.ai/models/claude-fable-5-1-high) · [xhigh](https://artificialanalysis.ai/models/claude-fable-5-1-xhigh) · [max](https://artificialanalysis.ai/models/claude-fable-5-1)
  - Haiku 4.5: [reasoning](https://artificialanalysis.ai/models/claude-4-5-haiku-reasoning) · [non-reasoning](https://artificialanalysis.ai/models/claude-4-5-haiku)
  - Sonnet 5.5: [low](https://artificialanalysis.ai/models/claude-sonnet-5-5-low) · [medium](https://artificialanalysis.ai/models/claude-sonnet-5-5-medium) · [high](https://artificialanalysis.ai/models/claude-sonnet-5-5-high) · [xhigh](https://artificialanalysis.ai/models/claude-sonnet-5-5-xhigh)
- [Artificial Analysis Intelligence Index 산정 방식](https://artificialanalysis.ai/methodology/intelligence-benchmarking)
- 아키텍처 · 설계 벤치마크
  - [R2ABench — Requirement-to-Architecture (arXiv 2604.06683)](https://arxiv.org/html/2604.06683v1)
  - [SWE-Lancer (arXiv 2502.12115)](https://arxiv.org/html/2502.12115v4)
  - [SAKE — Software Architectural Knowledge Evaluation (arXiv 2606.29520)](https://arxiv.org/html/2606.29520v1)
  - [SADU — Software Architecture Diagram Understanding (arXiv 2604.04009)](https://arxiv.org/html/2604.04009v1)
- 작업 연속성 · 오케스트레이션
  - [Vending-Bench 2 — Andon Labs](https://andonlabs.com/evals/vending-bench-2)
  - [Artificial Analysis: Opus 5.5 분석 (AA-Briefcase)](https://artificialanalysis.ai/articles/claude-opus-5-5) · [Sonnet 5.5 분석](https://artificialanalysis.ai/articles/claude-sonnet-5-5)
  - [Artificial Analysis: Terminal-Bench 4.0 리더보드](https://artificialanalysis.ai/evaluations/terminalbench-4-0)
  - [METR: Claude Opus 5.5 사전 평가 요약](https://metr.org/blog/2026-09-22-claude-opus-5-5/)
  - [MCP Atlas — Scale AI 리더보드](https://labs.scale.com/leaderboard/mcp_atlas)
  - [OrchestraBench (arXiv 2608.05263)](https://arxiv.org/html/2608.05263v1) · [OrchBench (arXiv 2607.25656)](https://arxiv.org/html/2607.25656v1)
  - [Claude Platform: Multiagent orchestration](https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration)
- 읽었지만 본문 근거로 쓰지 않은 자료 (횟수가 적은 개인 실험)
  - [Claude Opus 5.5 vs. Fable 5.1 — The New Stack](https://thenewstack.io/claude-opus-5-5-vs-fable-5-1/) (같은 작업 5회 비교)
  - [claude-code-orchestration-benchmark](https://github.com/Dealwatch/claude-code-orchestration-benchmark) (셀당 1회 케이스 스터디)
- Superpowers (v6.4.2)
  - [obra/superpowers](https://github.com/obra/superpowers)
  - [RELEASE-NOTES.md (v6.4.2, v6.4.1)](https://github.com/obra/superpowers/blob/main/RELEASE-NOTES.md)
  - [subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md)
  - [subagent-driven-development/implementer-prompt.md](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/implementer-prompt.md)
  - [writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md)
  - [brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md)
  - [requesting-code-review/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/requesting-code-review/SKILL.md)
  - [test-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/test-driven-development/SKILL.md)
  - [using-superpowers/references/claude-code-tools.md](https://github.com/obra/superpowers/blob/main/skills/using-superpowers/references/claude-code-tools.md)
- [codefoundry-io/triad-dispatch](https://github.com/codefoundry-io/triad-dispatch)

[^effort-field]: Claude Code 공식 문서 [Subagents](https://code.claude.com/docs/en/sub-agents), Configuration 표의 `effort` 항목.
[^full-conversation]: Claude Code 공식 문서 [Manage costs effectively](https://code.claude.com/docs/en/costs), "Why usage climbs in a long session" 절.
[^turn-count]: Superpowers v6.4.2 [subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md), "Model Selection" 절.
[^unattended]: Claude Platform 문서 [Prompting Claude Opus 5.5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5), "Unattended agentic runs" 절.
[^auto-delegation]: Claude Code 공식 문서 [Subagents](https://code.claude.com/docs/en/sub-agents), "Understand automatic delegation" 절.
[^opus-page]: Anthropic, [Introducing Claude Opus 5.5](https://www.anthropic.com/claude-opus-5-5) (2026-09-22). 벤치마크 표와 각주, 본문의 effort별 비용 비교. 2026-10-01 확인.
[^sonnet-page]: Anthropic, [Introducing Claude Sonnet 5.5](https://www.anthropic.com/claude-sonnet-5-5) (2026-09-28). 벤치마크 표와 각주 2 "Sonnet 5.5 scores lower at Max effort than at Xhigh". 2026-10-01 확인.
[^effort-names]: Claude Platform 문서 [Prompting Claude Opus 5.5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5), "Calibrate effort" 절: "Effort level names don't correspond to the same amount of thinking across models."
[^fable-prompting]: Claude Platform 문서 [Prompting Claude Fable 5.1](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1), "Consider all effort levels" · "Finish the whole task" · "Keep changes and tests to what the task asks for" · "Let the lead agent keep working while subagents run" 절. 2026-10-01 확인.
[^choosing]: Claude Platform 문서 [Choosing a model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model), "Option 2: Start capability-first" 절과 모델 선택 표. 2026-10-01 확인.
[^opus-sc]: Anthropic, [Claude Opus 5.5 System Card](https://www-cdn.anthropic.com/fc1b44717c85dc068bc6ba5024219938094694bd/Claude%20Opus%205.5%20System%20Card.pdf) (2026-09-22). 8장 Table 8.1.A와 8.3 DeepSWE · 8.4 FrontierCode · 8.7 FrontierSWE, 2.3절 내부 사용 관찰과 METR 외부 평가.
[^fable-sc]: Anthropic, [Claude Fable 5.1 & Claude Mythos 5.1 System Card](https://www-cdn.anthropic.com/0339e6a7c5c7b87f5c07798616dc32c215d14235/Claude%20Fable%205.1%20&%20Claude%20Mythos%205.1%20System%20Card.pdf) (2026-09-01). 8.3 DeepSWE · 8.4 FrontierCode · 8.5 FrontierSWE · 8.13 Multi-Agent, 생물 분야 평가의 약점 서술.
[^cost-intel]: Claude Platform 문서 [Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence), advisor · orchestrator 전략의 측정 예시. 2026-10-01 확인.
[^goal]: Claude Code 공식 문서 [Keep Claude working toward a goal](https://code.claude.com/docs/en/goal), "Set a goal" · "How evaluation works" 절.
[^mcp-atlas]: Scale AI, [MCP Atlas 리더보드](https://labs.scale.com/leaderboard/mcp_atlas). 2026-10-01 확인.
[^attention]: Vaswani et al., "Attention Is All You Need" (2017), Table 1: Self-Attention complexity per layer $O(n^2 \cdot d)$. [arXiv:1706.03762](https://arxiv.org/abs/1706.03762)
