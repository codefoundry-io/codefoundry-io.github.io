# 테마 커스터마이징 기록

Chirpy 테마(gem)에 없는 스타일을 이 저장소에서 덧붙인 내역. `_config.yml` 의
`exclude:` 에 `docs` 가 들어 있어 이 파일은 사이트에 발행되지 않는다.

Chirpy 는 RubyGem 으로 설치되므로 `_layouts`, `_includes`, `_sass` 원본은 이
트리에 없다. 원본을 보려면:

```bash
bundle info --path jekyll-theme-chirpy
```

---

## 1. `<details>` / `<summary>` 접이식 블록 스타일

- **파일**: `assets/css/jekyll-theme-chirpy.scss`
- **적용일**: 2026-08-16
- **테마 버전**: `jekyll-theme-chirpy` 7.6.0
- **적용 범위**: 사이트 전체 (모든 글의 `<details>`)

### 왜 넣었나

`2026-08-16-ai-terms-before-we-start` 글에서 이론과 실험 자료를 `<details>` 로
접어 본문 흐름을 방해하지 않게 했다. 그런데 로컬에서 확인해보니 **접힌 상태의
`<summary>` 가 바로 아래 문단과 붙어 보여서, 눌러서 펼치는 요소로 읽히지
않았다.**

원인을 gem 에서 확인한 결과 **Chirpy 는 `details` / `summary` 에 대한 스타일
규칙을 하나도 갖고 있지 않다.**

```bash
# vendors(부트스트랩 리셋)를 빼면 결과 없음
rg 'details|summary' "$(bundle info --path jekyll-theme-chirpy)/_sass"
```

즉 브라우저 기본 스타일로 렌더되며, 기본값에는 여백도 테두리도 hover 도 없다.
접이식 블록을 계속 쓸 것이므로 글마다 우회하지 않고 테마 레벨에서 해결했다.

### 어떤 효과인가

| 항목 | 이전 (브라우저 기본) | 이후 |
| :--- | :--- | :--- |
| 위아래 여백 | 없음 — 앞뒤 문단에 붙음 | `1.5rem` |
| 경계 | 없음 | 1px 테두리 + 왼쪽 3px 액센트 |
| 배경 | 본문과 동일 | `--card-bg` 패널 |
| hover | 없음 | 배경색 변화 (클릭 가능 신호) |
| 펼친 상태 | 제목/본문 구분 없음 | 제목 아래 구분선 |
| 제목 | 본문과 같은 굵기 | `font-weight: 600` + `--heading-color` |

접힌 상태에서는 얇은 한 줄 카드로 보이고, 펼치면 제목과 내용이 구분선으로
나뉜다.

### 다크 모드 / 라이트 모드

**하드코딩된 색이 하나도 없다.** 전부 Chirpy 테마 변수만 사용했고, 사용한 변수는
모두 `_sass/themes/_light.scss` 와 `_sass/themes/_dark.scss` **양쪽에** 정의되어
있음을 확인했다.

| 변수 | 라이트 | 다크 | 용도 |
| :--- | :--- | :--- | :--- |
| `--card-bg` | `white` | `#1e1e1e` | 블록 배경 |
| `--main-border-color` | `#f3f3f3` | `rgb(44 45 45)` | 테두리 · 구분선 |
| `--card-hover-bg` | `#e2e2e2` | `#464d51` | hover 배경 |
| `--heading-color` | `#2a2a2a` | `#cccccc` | 제목 글자색 |
| `--prompt-info-icon-color` | `#0070cb` | `#0075d1` | 왼쪽 액센트 |

왼쪽 3px 액센트를 넣은 이유: **라이트 모드에서는 `--card-bg` 가 `white` 라 본문
배경과 같고, `--main-border-color` 도 `#f3f3f3` 로 매우 흐리다.** 테두리만으로는
블록이 거의 안 보인다. 액센트 컬러는 양쪽 테마에서 모두 선명하게 보이므로 이걸로
블록의 시작점을 잡았다.

`--prompt-info-icon-color` 를 고른 것은 이 글에서 `<details>` 바로 위에
`prompt-info` 박스로 "눌러 펼쳐보자" 안내를 두었기 때문이다. 안내 박스와 접이식
블록이 같은 계열 색으로 묶여 하나의 단위로 읽힌다.

`prefers-reduced-motion: reduce` 인 환경에서는 hover transition 을 끈다.

### 주의 — 파일을 수정할 때

`assets/css/jekyll-theme-chirpy.scss` 는 gem 의 동명 파일을 **덮어쓰는** 것이다.
파일 상단의 아래 블록은 gem 원본 그대로이며 **절대 지우면 안 된다.**

```scss
@use 'abstracts/variables' with (
  $theme: '{{ site.theme_mode }}'
);

/* prettier-ignore */
@use 'main
{%- if jekyll.environment == 'production' -%}
  .bundle
{%- endif -%}
';
```

- `$theme: '{{ site.theme_mode }}'` — `_config.yml` 의 다크/라이트 고정 설정을
  전달한다. 빠지면 테마 모드 설정이 동작하지 않는다.
- `main.bundle` 분기 — production 빌드에서만 번들 스타일을 쓴다. 빠지면
  프로덕션 CSS 가 달라진다.

커스텀 스타일은 `/* append your custom style below */` 아래에만 추가한다.

테마를 업그레이드하면 이 파일이 gem 원본과 어긋날 수 있으므로, 위 블록을
gem 쪽과 다시 대조할 것.

### 검증

```bash
./tools/test.sh   # production 빌드 + html-proofer
./tools/run.sh    # 로컬 서버에서 라이트/다크 양쪽 눈으로 확인
```

다크 모드는 사이드바 하단의 모드 토글로 전환해 확인한다.

---

## 2. `.user-prompt` — 사용자가 AI에게 입력한 프롬프트

- **파일**: `assets/css/jekyll-theme-chirpy.scss`
- **적용일**: 2026-08-16
- **적용 범위**: 사이트 전체 (`{: .user-prompt }` 를 붙인 인용문)

### 왜 넣었나

이 블로그는 AI에게 던진 프롬프트를 자주 인용한다. 그런데 인용문(`>`)으로 적으면
**세 가지가 전부 같은 모양이 된다.**

1. 사용자가 AI에게 입력한 프롬프트 — `코드를 수정해.`
2. AI가 한 말
3. 공식 문서 인용 · 글쓴이의 방백

`2026-08-16-ai-terms-before-we-start` 글에는 1번이 6개 있는데, 독자가 "이건 저자가
실제로 타이핑한 문장"이라는 걸 알 방법이 없었다. Chirpy 의 `prompt-info` 색 박스와
이름까지 헷갈린다.

### 어떤 효과인가

| | 일반 인용문 (`>`) | `.user-prompt` |
| :--- | :--- | :--- |
| 왼쪽 | 얇은 세로선 | 큰 여는 따옴표 `"` |
| 배경 | 없음 | 옅은 틴트 |
| 글자색 | `--blockquote-text-color` (회색) | `--heading-color` (본문 농도) |
| 읽히는 방식 | 흘려 읽는 인용 | 입력한 문장 |

글자 농도를 본문 수준으로 되돌린 것이 핵심이다. 일반 인용문은 회색이라 시선이
스쳐 지나가지만, 프롬프트는 글의 논지를 끌고 가는 주체이므로 또렷해야 한다.

`프레이밍(Framing)` 섹션에 이 대비가 그대로 드러난다. 사용자 프롬프트(`너는
고양이야...`)는 틴트 블록이고, 바로 아래 AI 발화(`너는 되도 않는 짓을...`)는 회색
인용문으로 남아 두 화자가 눈으로 구분된다.

### 클래스 이름 규칙 — 중요

**이름이 `prompt-` 로 시작하면 안 된다.** Chirpy 는 다음 셀렉터로 색 박스를
입힌다.

```scss
blockquote[class^='prompt-'] {
  border-left: 0;
  padding: 1rem 1rem 1rem 3rem;   /* 왼쪽 3rem = 아이콘 자리 */
  &::before { /* Font Awesome 아이콘 */ }
}
```

`prompt-user` 로 지었다면 이 규칙에 걸려 의도한 가벼운 스타일이 덮인다. 그래서
`user-prompt` 로 뒤집어 지었다.

### 다크 모드 / 라이트 모드

| 변수 | 라이트 | 다크 | 용도 |
| :--- | :--- | :--- | :--- |
| `--inline-code-bg` | `rgb(25 25 28 / 5%)` | `rgb(255 255 255 / 5%)` | 배경 틴트 |
| `--heading-color` | `#2a2a2a` | `#cccccc` | 본문 글자색 |
| `--text-muted-color` | `#757575` | `#868686` | 따옴표 색 |

`--inline-code-bg` 를 고른 이유: **불투명 색이 아니라 배경 위에 얹는 5% 반투명
틴트**다. 라이트에서는 어두운 틴트, 다크에서는 밝은 틴트로 정의되어 있어 어느
쪽에서든 "바탕보다 살짝 다른 면"이 된다. 밝기를 직접 지정할 필요가 없다.

브라우저에서 계산된 값으로 양쪽을 확인했다.

| | 라이트 | 다크 |
| :--- | :--- | :--- |
| 배경 | `rgba(25, 25, 28, 0.05)` | `rgba(255, 255, 255, 0.05)` |
| 글자 | `rgb(42, 42, 42)` | `rgb(204, 204, 204)` |

### 사용 규칙

`.claude/skills/chirpy-blog-decorate/` 에도 같은 내용을 넣어두었다.

- **쓸 때**: 저자가 AI에게 입력한 짧은 지시문
- **쓰지 말 것**: AI의 답변, 공식 문서 인용, 저자의 방백 — 일반 `>` 로 둔다.
  대비가 생기는 것이 이 클래스의 목적이다

---

## 3. 표 — 좁은 화면에서 자동 줄바꿈

- **파일**: `assets/css/jekyll-theme-chirpy.scss`
- **적용일**: 2026-08-16
- **적용 범위**: 768px 미만 화면의 모든 표

### 왜 넣었나

문장이 들어가는 표가 폰에서 가로로 밀려 읽을 수 없었다. 원인은 테마가 모든
셀에 거는 `white-space: nowrap` 이다.

```scss
/* _sass/abstracts/_placeholders.scss */
%table-cell {
  padding: 0.4rem 1rem;
  font-size: 95%;
  white-space: nowrap;
}
```

셀이 절대 접히지 않으므로 표 폭이 내용 길이만큼 무한정 늘어나고,
`.table-wrapper` 의 `overflow-x: auto` 가 그걸 가로 스크롤로 밀어낸다.

### 데스크탑은 왜 그대로 두는가

`nowrap` 은 실수가 아니라 의도된 선택이다. 짧은 데이터 표에서는 셀이 제멋대로
접히지 않는 편이 낫고, `gemma-4-26b-a4b-it@q8_k_xl` 같은 식별자가 중간에서
끊기면 오히려 읽기 어렵다.

그래서 **데스크탑에서는 테마 기본 동작을 유지하고, 줄바꿈 위치는 본문에서
`<br>` 로 직접 지정한다.** 절 경계에서 끊을 수 있어 자동 줄바꿈보다 가독성이
좋다. 폰은 그런 수동 조정이 통하지 않을 만큼 좁으므로 자동으로 처리한다.

| | 데스크탑 (≥768px) | 폰 (<768px) |
| :--- | :--- | :--- |
| `white-space` | `nowrap` (테마 기본) | `normal` |
| 줄바꿈 | 본문의 `<br>` 로 수동 지정 | 자동 |
| 넘칠 때 | 가로 스크롤 | 없음 (폭에 맞춰 접힘) |

### 셀렉터 명시도 — 빠지기 쉬운 함정

**미디어 쿼리는 명시도를 올려주지 않는다.** 처음에 이렇게 썼다가 규칙이
그대로 무시됐다.

```scss
.table-wrapper > table td { white-space: normal; }   /* (0,1,2) */
```

테마 규칙이 `.table-wrapper > table tbody tr td` 로 **(0,1,4)** 라서 미디어
쿼리 안에 있어도 이쪽이 이긴다. 셀렉터를 테마와 같은 형태로 맞추고 파일
뒤쪽에 두어야 적용된다.

```scss
.table-wrapper > table {
  thead th,
  tbody tr td { white-space: normal; }               /* (0,1,3) / (0,1,4) */
}
```

`!important` 를 쓰지 않은 이유다. 명시도만 맞추면 충분하다.

### 효과

iPhone 폭(390px, 컨테이너 307px)에서 측정한 값이다.

| 표 | 이전 | 이후 |
| :--- | ---: | ---: |
| 설정키 / 설명 / 값 | 687px (초과 +380) | **307px** |
| 영향받는 요소 / 원인 | 500px (초과 +193) | **307px** |
| 위치 / hello / hi | 724px (초과 +417) | **307px** |
| 도구 / 지침 파일 / … | 702px (초과 +359) | **343px** |

네 표 모두 가로 스크롤이 사라졌다.

`table-layout` 은 건드리지 않았다. 브라우저 자동 배치가 내용이 긴 열에 폭을
더 주므로 `fixed` 로 균등 분배하는 것보다 결과가 낫다. `overflow-wrap:
anywhere` 는 긴 코드나 URL 이 셀을 뚫고 나가는 것만 막는다.
