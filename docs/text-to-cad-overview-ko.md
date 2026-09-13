# text-to-cad 분석 정리 (한국어)

이 문서는 `text-to-cad` 레포지토리를 처음 받아본 뒤, 무엇을 하는 도구인지 /
언제 쓰는지 / 어떻게 활용하고 수익화할 수 있는지 조사·분석한 내용을 정리한
노트다. 레포의 기능 문서가 아니라 **학습·검토 기록**이다.

## 관련 주소

| 구분 | 주소 |
| --- | --- |
| 원본 저장소 (upstream) | https://github.com/earthtojake/text-to-cad |
| 이 저장소 (fork) | https://github.com/bmshin94/text-to-cad |
| 공식 문서 사이트 | https://www.texttocad.dev |
| 커뮤니티 | https://discord.gg/5FGB9DwJYU |

분석 시점 기준: `VERSION` = 0.5.1, 라이선스 MIT (Copyright (c) 2026 Thompson Labs LLC)

---

## 1. 한 줄 요약

**자연어·이미지로 설명하면 실제 제조 가능한 CAD 파일을 만들어주는 AI 에이전트
스킬 라이브러리.** 프로그램이 아니라, AI 에이전트에게 CAD/CAE/CAM 작업 능력을
부여하는 지침서 + 실행 엔진 묶음이다.

기존 Text-to-3D 도구들과 결정적으로 다른 점:

- 결과물이 메시(삼각형 덩어리)가 아니라 **STEP** — 산업 표준 CAD 교환 포맷
- **파라메트릭** — 파이썬 상수만 바꾸면 치수가 바뀐다
- 만든 뒤 **검사·검증·수정 루프**가 존재한다

---

## 2. 저장소 구조

| 경로 | 역할 |
| --- | --- |
| `skills/` | **핵심 산출물.** 11개 에이전트 스킬 (마크다운 지침서 + 스크립트) |
| `packages/cadgen/` | 실제 엔진. PyPI 배포용 파이썬 패키지 (생성·검사·뷰어 백엔드·빌드 데몬) |
| `packages/cadgen-js/` | 공용 JS 렌더/런타임 (three.js 기반, UI 프레임워크 중립) |
| `apps/viewer/` | CAD Viewer의 React 클라이언트 |
| `apps/docs/` | 문서 사이트 |
| `models/` | 예제·고정 fixture 모음 (Git LFS) |
| `tests/`, `scripts/` | 테스트 스위트, 레포 상시 명령 |
| `.claude-plugin/`, `.codex-plugin/` | 에이전트 플러그인 매니페스트 |

### 스킬 11종

| 스킬 | 역할 |
| --- | --- |
| `cad` | 핵심. 자연어/이미지/도면 → 파라메트릭 3D (STEP, STL, 3MF, GLB) |
| `cad-viewer` | 로컬 브라우저 3D 리뷰 (키네매틱스·애니메이션·측정) |
| `dxf` | 2D 도면 (가스켓, 판금 전개도, 레이저 컷 레이아웃) |
| `step-parts` | 나사·베어링·모터 등 기성품 STEP 검색·다운로드 |
| `urdf` / `srdf` / `sdf` | 로봇 구조 / MoveIt2 플래닝 / 시뮬레이터 설명 파일 |
| `dfam-check` | 3D프린팅 가능성 검사 (벽두께·오버행·서포트·빌드 방향) |
| `gcode` | 메시 → 검증된 FDM `.gcode` (실제 슬라이서 CLI 오케스트레이션) |
| `bambu-labs` | Bambu Lab 프린터 LAN 업로드·출력 시작 |
| `sendcutsend` | SendCutSend 발주 전 DXF/STEP 사전 검증 |

### 파이프라인이 끝까지 이어진다

```
자연어 설명 → 3D 모델(STEP) → 출력 가능성 검사 → G-code → 프린터 출력
```

### models/ 의 데모 규모

`w16`(부가티 W16 8.0L 쿼드터보 엔진, 크랭크 회전 애니메이션 포함),
`falcon_heavy`(팰컨 헤비 재현), `tendon_hand`(힘줄 구동 로봇 손),
`f1`, `hypercar`, `moonwatch`, `motorbike`, `qdd_actuator`,
`juno`/`lyra`(URDF/SRDF 로봇 패키지), `examples`(학습용 기본 부품).

### 설계 철학: "설계를 코드로"

```python
from cadgen import build123d as bd
from cadgen import step

WIDTH = 10.0

@step                      # 데코레이터는 선언만 한다
def bracket():
    return bd.Box(WIDTH, 10, 10)

if __name__ == "__main__":
    bracket()              # 호출이 빌드를 수행한다
```

상수만 바꾸면 치수가 바뀌고, 결과가 파이썬 소스이므로 Git으로 버전 관리·diff·
리뷰가 가능하다. 바이너리 CAD 파일로는 불가능한 부분이다.

---

## 3. 설치 및 사용법

설치는 **2단계**로 나뉜다.

### 1단계 — 스킬(지침서) 설치

```bash
# ① Skills CLI (권장)
npx skills add earthtojake/text-to-cad

# ② Claude Code 플러그인
claude plugin marketplace add earthtojake/text-to-cad
claude plugin install cad@text-to-cad

# ③ Codex / Grok
codex plugin marketplace add earthtojake/text-to-cad
codex plugin add cad@text-to-cad
grok plugin install earthtojake/text-to-cad --trust
```

> 업데이트도 `add`를 다시 실행한다. `npx skills update`는 lockfile에 이미 있는
> 스킬만 갱신하므로 **새로 추가된 스킬을 놓친다.**

### 2단계 — 엔진(cadgen) 설치

```bash
# Python 3.11+ 필요
python -m pip install -r requirements.txt
python -m playwright install chromium   # 스냅샷 렌더링용 브라우저
```

의존성: `build123d`, `cadquery-ocp`(OpenCascade 바인딩), `ezdxf`, `shapely`.
렌더링은 optional extra(`snapshot` → playwright).

> **이 레포를 직접 개발할 때는 `requirements-dev.txt`를 써야 한다.**
> 스킬의 `requirements.txt`는 `cadgen==<VERSION>`을 PyPI에서 받아오므로,
> 체크아웃의 수정 사항이 아니라 배포판이 설치된다.
> `requirements-dev.txt`는 `--editable ./packages/cadgen`으로 연결한다.

### 주요 명령

```bash
python src/l_bracket.py                  # 모델 스크립트 실행 → STEP 생성
cadgen viewer                            # 로컬 3D 뷰어
cadgen step inspect STEP/l_bracket.step  # 치수·구조 검사
cadgen store why src/l_bracket.py        # 왜 stale/current 인지 설명
cadgen doctor skills/cad                 # 설치된 cadgen ↔ 스킬 핀 일치 확인
```

실제로는 이 명령들을 사람이 직접 칠 일은 적다. 에이전트가 스킬 지침에 따라 호출한다.

### 주의

이 도구는 **로컬 실행이 전제**다. 뷰어는 로컬 브라우저에서 열리고, 프린터는
같은 LAN 안에 있어야 한다. `models/`는 LFS 포인터로 오므로 필요할 때만 hydrate한다.

---

## 4. 스킬 / 플러그인 / MCP 구분

**본질은 스킬(Skill)이다. 플러그인은 배포 포장지이고, MCP는 쓰지 않는다.**
(레포 전체를 검색해 확인했다. "MCP"로 검색되는 것은 로봇 손 모델의 MCP 관절
= 중수지관절이다.)

| 구분 | 정의 | 특징 |
| --- | --- | --- |
| Skill | 에이전트가 **읽는** 마크다운 + 스크립트 | 컨텍스트에 로드되는 지식/절차 |
| MCP | 에이전트가 **호출하는** 별도 서버 | 프로토콜·서버 프로세스 필요 |
| Plugin | 위를 묶어 배포하는 매니페스트 | `.claude-plugin/`, `.codex-plugin/` |

플러그인으로 깔든 Skills CLI로 깔든 **내용물은 같은 `skills/` 폴더**다.

### MCP를 쓰지 않은 이유

이미 `cadgen` CLI가 있으므로 에이전트는 Bash로 직접 실행하면 된다. MCP로
감쌀 이유가 없고, MCP는 도구 목록을 항상 노출해야 하는 반면 스킬은 필요할 때만
읽는다(**progressive disclosure**).

이 레포는 그 패턴의 교과서적 예시다.

- 평소: `description` 한 줄만 본다 (트리거 판단용)
- CAD 작업 시작: `SKILL.md`를 읽는다
- 조립·위치 작업 차례: `references/positioning.md`만 추가로 읽는다

---

## 5. API 토큰 필요 여부

**이 도구 자체는 어떤 API 토큰도 요구하지 않는다.**

| 대상 | 인증 | 비고 |
| --- | --- | --- |
| cadgen 엔진 | 불필요 | 로컬 파이썬, 오프라인 동작 |
| CAD Viewer | 불필요 | 로컬 서버, 호스팅 배포 없음 |
| step.parts | 불필요 | 공개 API (`api.step.parts`). 네트워크만 필요 |
| SendCutSend | 불필요 | 발주가 아니라 검증만 수행 |
| Bambu 프린터 | **필요** | API 토큰이 아니라 프린터 IP + LAN 액세스 코드 |

Bambu 설정은 작업 폴더 루트의 `bambu-printers.json`에 저장하며, 스킬 문서가
**Git에 커밋하지 말라고 명시**한다. 클라우드를 거치지 않고 LAN 내부에서만 통신한다.

따라서 발생하는 비용은 **LLM 호출 비용뿐**이다. CAD 연산은 로컬 CPU가 수행한다.

---

## 6. GitHub에서 인기 있는 이유

분석 시점 기준: ⭐ 약 15,500 / fork 약 1,600 / watcher 87.
Topics: `ai-agents`, `cad`, `robotics`, `mechanical-engineering`, `step`, `stl`.

1. **"AI로 CAD"는 다들 실패한 영역이었다.** 기존 생성형 3D는 메시만 나와 제조가
   불가능하고 치수 수정도 안 됐다. 이 레포는 STEP + 파라메트릭 + 검증 루프로
   "실제로 제조 가능한 결과"를 낸다.
2. **스킬 설계의 교과서.** `AGENTS.md`에 법칙을 명시하고(스킬 간 import 금지 등),
   지침은 얇게 / 심화 문서는 분리하고, 버전 핀 + `cadgen doctor`로 문서-코드
   불일치를 감지한다. 개발자들이 "스킬 만드는 법"을 배우려고 본다.
3. **데모의 임팩트.** W16 엔진, 팰컨 헤비, 로봇 손을 README 최상단 GIF로 배치.
4. **실물까지 이어진다.** 설계에서 끝나지 않고 프린터에서 물건이 나온다.
5. **로봇 붐 타이밍.** URDF/SRDF/MoveIt2 지원.
6. **메이커/3D프린터 커뮤니티**라는 크고 열정적인 사용자층.
7. **벤더 중립.** Claude Code, Codex, Grok 모두 지원.
8. **MIT + 완전 로컬 실행.** 기업도 설계 유출 우려 없이 도입 가능.

---

## 7. 로컬 에이전트 구축에 참고할 점

도메인이 달라도(법률·회계·의료 등) 뼈대를 그대로 차용할 수 있다.

1. **`description` 작성법.** 사용자가 쓸 만한 단어를 모두 나열해야 스킬이
   발동한다. "쓰지 말아야 할 경우"도 명시한다.
2. **로직을 스킬에 넣지 않는다.** 스킬은 얇은 entrypoint, 실제 로직은 설치형
   패키지(`cadgen`). 버전을 핀으로 고정하고 `cadgen doctor`로 드리프트를 잡는다.
3. **스킬 간 독립.** 서로 import하지 않고, 명시적 인계(handoff)로 연결한다
   (예: STEP을 만들었으면 반드시 `$cad-viewer`에 경로를 넘긴다).
4. **검증 루프.** 만들기 → 검사 → 스냅샷 → 확인 → 실패 시 `repair-loop.md`.
   에이전트가 자기 결과를 확인하게 만드는 것이 신뢰의 핵심이다.
5. **되돌릴 수 없는 동작은 신중하게.** 프린터 출력은 `--dry-run` 먼저, 확인 후 실행.
6. **경계선 문서화.** 각 패키지 README에 "MAY DEPEND ON / DEPENDED ON BY"를 적어
   에이전트가 수정 범위를 헷갈리지 않게 한다.
7. **릴리스 자동화.** `VERSION` 하나를 단일 진실 공급원으로 두고, 손으로 고치는
   것을 CI가 막는다.

읽는 순서 권장: `AGENTS.md` → `skills/cad/SKILL.md` →
`skills/step-parts/SKILL.md`(70줄, 구조 파악용) →
`skills/cad/references/repair-loop.md` → `packages/cadgen/README.md`.

---

## 8. 수익화 검토

### 8.1 시장 현실

Text-to-CAD는 이미 경쟁이 있는 시장이다.

| 서비스 | 가격 | 특징 |
| --- | --- | --- |
| Zoo.dev (구 KittyCAD) | 사용량 과금 (~$0.0083/초) | 카테고리 원조, API 성숙 |
| AdamCAD | $5.99/월부터 | YC W25, $4.1M 투자, 파라메트릭 슬라이더 |
| Leo AI | $39/월 ~ $1,800/년 | 엔지니어링 특화, 다부품 조립 |
| Spectral Labs (SGS-1), CADGPT | — | 신생/주요 플레이어로 언급 |

국내 제조 중개는 **[캐파(CAPA)](https://capa.ai/)**가 이미 자리를 잡고 있다.
제조 파트너 약 2,700개(공정 합산), CNC·3D프린팅·금형사출·판금·주조·전자회로
등을 다루고, 도면 기반 협업 SaaS '캐파 커넥트'도 운영한다.
→ **중개 정면승부는 현실적이지 않다.**

### 8.2 차별화 가능한 축

1. **원가 0** — MIT 라이선스로 검증된 엔진을 공짜로 쓴다. 경쟁사가 수년·수십억
   들인 부분이 출발선에서 사라진다.
2. **검증 레이어** — `dfam-check`, `sendcutsend`, `cadgen step inspect`.
   경쟁사 대부분은 "생성"만 한다. "만들 수 있는 물건인가"를 보장하는 것이 해자다.
3. **로봇(URDF/SRDF)** — 위 경쟁사 중 로봇을 다루는 곳이 없다. 가장 빈 영역.
4. **한국어 + 국내 공급망** — KS 규격, 국내 유통사, 한국 결제·CS.

### 8.3 단위 원가 추정

실제 API 단가:

| 모델 | 입력 $/1M | 출력 $/1M |
| --- | --- | --- |
| Claude Opus 5 | $5.00 | $25.00 |
| Claude Sonnet 5 | $2.00 | $10.00 |
| Claude Haiku 4.5 | $1.00 | $5.00 |

할인 수단: 프롬프트 캐싱(캐시 읽기 = 입력가의 0.1배, 쓰기 1.25배),
배치 API(비동기 50% 할인).

이 레포 기준 토큰량 (실측한 문서 크기):

| 파일 | 크기 | 추정 토큰 |
| --- | --- | --- |
| `skills/cad/SKILL.md` | 28,299자 | ~7,100 |
| `references/build123d-modeling.md` | 25,982자 | ~6,500 |
| `references/step-generation.md` | 24,965자 | ~6,200 |
| `references/positioning.md` | 13,851자 | ~3,500 |
| `references/inspection-and-validation.md` | 15,295자 | ~3,800 |

→ 작업 1건당 지침서만 약 17,000 토큰 (SKILL.md + 참조 2개)

**생성 1건 원가 (Sonnet 5, 8턴 반복, 출력 10k 토큰, 캐싱 적용, 환율 1,400원 가정):**

| 항목 | 계산 | 비용 |
| --- | --- | --- |
| 캐시 쓰기 | 17k × 1.25 × $2/M | $0.043 |
| 캐시 읽기 | 200k × 0.1 × $2/M | $0.040 |
| 신규 입력 | 16k × $2/M | $0.032 |
| 출력 | 10k × $10/M | $0.100 |
| **합계** | | **≈ $0.21 (약 290원)** |

| 작업 난이도 | Sonnet 5 | Opus 5 |
| --- | --- | --- |
| 간단 (브래킷, 거치대) | 약 290원 | 약 730원 |
| 중간 (구멍·필렛 다수) | 약 800원 | 약 2,000원 |
| 복잡 (조립품, 반복 수정 많음) | 약 3,000원 | 약 7,500원 |

> 추정치다. 실제 원가는 repair loop 반복 횟수가 좌우하므로, 몇 건 돌려보고
> `usage` 로그로 검증해야 한다.

**결론: 무제한 요금제는 성립하지 않는다.** 월 8,400원(AdamCAD 수준)에 50건을
생성하면 원가만 14,500원이다. 크레딧/건수 제한, 모델 라우팅(간단한 건 Haiku/
Sonnet), 캐싱 필수, 실패 건 미과금이 전제 조건이다.

### 8.4 아이디어 비교

| 아이디어 | 첫 매출 | 난이도 | 마진 | 비고 |
| --- | --- | --- | --- | --- |
| ① 기업 커스텀 스킬 컨설팅 | 1~2개월 | 낮음 | 매우 높음 | 국내 AI 에이전트 프로젝트 단가: 프로토타입 1천~5천만원, 전사 도입 수억 |
| ② 로봇(URDF/SRDF) SaaS | 4~8개월 | 높음 | 95%+ | 경쟁 공백 지대 |
| ③ 니치 제품 무한 변형 판매 | 2~4주 | 낮음 | 중간 | 차종별 거치대 등. 경쟁자는 손으로 그려야 함 |
| ④ 교육 콘텐츠 | 1~2개월 | 매우 낮음 | ~100% | 한국어 콘텐츠 공백. ①의 영업 도구 |
| ⑤ 캐파 "앞단" 공략 | 3~6개월 | 중간 | 90%+ | 3D 파일 없어 견적을 못 내는 고객을 잡는다 |
| ⑥ 한국판 부품 카탈로그 API | 6~12개월 | 매우 높음 | 높음 | 데이터가 해자지만 초기 투입이 큼 |
| ⑦ B2C 범용 SaaS | 3~6개월 | 높음 | 낮음 | Zoo/Adam/Leo와 정면승부. 용도 특화 필수 |

### 8.5 권장 순서

```
0~1개월    ④ 콘텐츠 시작(원가 0) + ③ 품목 10종 실물 테스트
1~4개월    ① 컨설팅 영업 (④가 포트폴리오) → 현금 확보 + 고객 문제 파악
4~10개월   ② 로봇 SaaS 또는 ⑦ 특화 SaaS 개발 (①의 매출로 버티기)
10개월+    ⑤ 제휴 / ⑥ 카탈로그로 확장
```

원칙: 제품을 먼저 만들지 않는다. 서비스로 돈을 벌면서 고객을 배우고 제품화한다.

### 8.6 피해야 할 것

- 무제한 생성 요금제 (확실한 적자)
- 캐싱 없는 서비스 (원가 2~3배)
- 모든 요청에 최상위 모델 사용
- Zoo/Adam과 범용 정면승부, 캐파와 중개 정면승부
- 안전 중요 부품(하중 지지·의료·항공·차량 주행부) 취급
- 무제한 무료 체험 (어뷰징 → 비용 폭탄)

---

## 9. 리스크 체크리스트

1. **제조물 책임.** 약관에 "출력물은 설계 초안이며 제조·사용 전 자격을 갖춘
   엔지니어의 검증이 필요하다"를 명시하고, 안전 중요 용도를 배제한다. 실물을
   판매하면 PL보험에 가입한다. 스킬 문서도 엔지니어링 인증·FEA 결론을 제공하지
   않는다고 명시하고 있으므로 그대로 반영하면 된다.
2. **라이선스.** MIT이므로 상업적 이용은 자유. 저작권 고지는 유지해야 한다.
3. **사업자 요건.** 실물 판매는 통신판매업 신고, SaaS는 PG 계약이 필요하다.
4. **API 비용 통제.** 사용자별 일/월 하드 캡, 이상 사용 차단, 예산 알림.

---

## 10. 기술 스택 관점 (React / PHP로 감쌀 수 있는가)

- **기하 커널은 재구현 대상이 아니다.** `cadquery-ocp` = OpenCascade(OCCT)
  바인딩으로, 1990년대부터 축적된 C++ 기하 커널이다. 그대로 재사용한다.
- **프론트엔드는 이미 React다.** `apps/viewer/`가 React + three.js 클라이언트이고,
  `packages/cadgen-js`는 의도적으로 프레임워크 중립으로 설계되어 재사용 가능하다
  (`three`, `three-mesh-bvh`, `meshoptimizer`).
- **PHP는 백엔드(회원·결제·주문·작업 큐)로 적합하다.** 기하 연산은 파이썬 워커에
  맡긴다.

권장 아키텍처:

```
[React 프론트]  프롬프트 입력 / 3D 미리보기(cadgen-js 재사용)
      │ REST
[PHP or Node]   회원 · 결제 · 주문 · 작업 큐
      │ queue
[Python 워커]   cadgen: LLM 호출 → 모델 스크립트 → STEP/STL → 검증
      │
[객체 스토리지]  GLB를 뷰어로 전달
```

브라우저 단독 실행이 필요하면 OCCT의 WASM 계열(replicad, opencascade.js) 또는
manifold, JSCAD를 검토할 수 있다. 서버 비용이 없고 설계가 브라우저를 벗어나지
않지만, 초기 로딩 용량과 기능·성능에 제약이 있다. 프로토타입에 적합하다.

---

## 11. 다음 단계 후보

1. 로컬 환경에 실제 설치 후 `models/examples`의 스크립트 실행 및 뷰어 확인
2. `AGENTS.md` + `skills/cad/SKILL.md` 정독 (스킬 설계 원리)
3. React 웹앱 프로토타입 설계
4. 수익화 아이디어 ① 또는 ② 상세 사업 계획
5. 자체 스킬 1종 직접 작성
