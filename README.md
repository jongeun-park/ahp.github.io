# 자원안보 진단·평가 지표 AHP 전문가 설문

에너지경제연구원 에너지안보정책연구실. GitHub Pages(설문 화면) + Supabase(응답 저장) 조합으로, 별도 서버 없이 동작합니다.

```
응답자 브라우저 ── index.html (GitHub Pages)
      │   쌍대비교 · CR 계산 · 가중치 산출은 모두 브라우저에서
      │   제출 시 JSON 한 건을 전송 (publishable 키, 행 추가 권한만)
      ▼
Supabase 프로젝트 KEEI (도쿄 리전)
      ├─ ahp_responses   응답 1건 = 1행 (원본)
      └─ v_weights · v_judgments · v_cr · v_group_weights   분석용 뷰
```

## 1. 파일

| 파일 | 역할 |
|---|---|
| `index.html` | 설문 페이지 전체. 설정·계산·화면·전송이 한 파일. GitHub에 올리는 것은 이것뿐 |
| `supabase.sql` | 테이블·권한·뷰 생성. **새 프로젝트를 처음부터 만들 때**만 실행 |
| `supabase_update.sql` | 응답자 정보를 소속·이름으로 바꾼 변경분. 현재 KEEI 프로젝트에는 이미 적용됨 |
| `supabase_update2.sql` | 뷰에서 쓰지 않는 `qualitative` 열 제거. 응답 행 유지, 반복 실행 가능 |
| `supabase_reset.sql` | 테스트 응답 삭제. **본 조사 시작 직전에 1회 실행**. 되돌릴 수 없음 |
| `README.md` | 이 문서 |

## 2. 설문 구성

안정성과 대응력, 두 개의 독립된 계층입니다. 두 계층 사이는 비교하지 않고, 각 계층 안에서 ① 평가영역 간 비교 → ② 영역별 세부지표 간 비교를 합니다.

| 단계 | `section_id` | 비교 대상 | 항목 | 쌍 |
|---|---|---|---|---|
| 1 | A0 | 안정성 평가영역 | 공급 · 수요 · 시스템 | 3 |
| 2 | A1 | 공급 세부지표 | 자원자립도 · 정치적 안정성 · 수입 편향성 · 생산 편향성 · 국내 공급 인프라 안정성 | 10 |
| 3 | A2 | 수요 세부지표 | 수입 안정성 · 가격 안정성 · 수요산업 규모 | 3 |
| 4 | A3 | 시스템 세부지표 | 정보 접근성 · 공공정책 및 제도 | 1 |
| 5 | B0 | 대응력 평가영역 | 상시 대응력 · 비상 대응력 · 인프라 | 3 |
| 6 | B1 | 상시 대응력 세부지표 | 수입원 다변화 · 외교 및 국제협력 · 자원 R&D · 자원 재활용 | 6 |
| 7 | B2 | 비상 대응력 세부지표 | 자원개발률 · 비축물량 · 위기대응 숙련도 | 3 |
| 8 | B3 | 인프라 세부지표 | 비상대응 시스템 · 인적자원 공급 | 1 |

총 30쌍, 약 12분.

- **척도**: 5점. Saaty 척도의 다섯 언어적 등급(1 동등 · 3 약간 · 5 더 · 7 매우 · 9 절대적으로)을 씁니다. 1~5 정수 척도로 바꾸는 방법은 `index.html`의 `CONFIG.scale` 주석에 있습니다.
- **일관성 검증**: 항목이 3개 이상인 단계는 응답 완료 즉시 고유벡터법으로 CR을 계산합니다. **CR > 0.2 → 오류 표시, 그 단계는 미완료로 유지**(수정해야 ✓), 0.1 < CR ≤ 0.2 → 주의 표시(완료 처리), 항목 2개인 단계(4·8)는 검증 대상 아님. 오류 시 가장 어긋나는 쌍을 붉은 바탕으로 표시만 하고, 어느 값으로 고치라는 안내는 하지 않습니다.
- **응답자 정보**: 소속과 이름. 둘 다 입력해야 시작됩니다.
- **단계 이동**: 8개 단계는 원하는 순서로 응답할 수 있습니다. 상단의 단계 이름을 누르면 어느 단계로든 이동하고, 완료된 단계(모든 쌍 응답 + CR ≤ 0.2)에는 ✓가 붙습니다. 결과·제출은 모든 단계가 완료되어야 열리며, 제출 후에는 이동이 막힙니다.
- **계층도**: 각 단계 상단에 지표 계층도를 그려 비교 중인 지표를 강조하고, 평가영역 비교 단계(1·5)에는 영역 정의를 함께 표시합니다. 그림은 설정의 지표 이름에서 자동 생성됩니다.
- **결과 화면**: 영역 가중치 × 영역 내 가중치 = 종합 가중치를 표로 보여준 뒤 제출. 계층별 세부지표 종합 가중치 합 = 100%.
- **임시저장**: 응답을 누를 때마다 응답자 브라우저(localStorage)에 자동 저장. 같은 기기·같은 브라우저로 다시 열면 "이어서 응답" 배너가 뜨고 첫 미완료 단계로 이동. 제출이 완료되면 삭제. 다른 기기나 사생활 보호 창에서는 복원되지 않음.
- **PDF 저장**: 결과 화면과 제출 완료 화면의 「PDF로 저장」은 브라우저 인쇄 대화상자를 열며, 대상에서 "PDF로 저장"을 고르면 응답자 정보·가중치표·쌍대비교 응답 30건이 담긴 문서가 저장됨. 제출 실패 시 이 PDF 또는 JSON 파일을 담당자에게 보내면 됨.
- **화면**: 검정 글씨, 각진 모서리, KoPubWorld 돋움(CDN 웹폰트, 미설치 PC에서도 동일). PC·모바일 모두 대응.

## 3. 현재 설정 상태

- Supabase 프로젝트: **KEEI** (`https://ocigorjuluhoylpnthbi.supabase.co`, Northeast Asia/Tokyo, 무료 플랜)
- `index.html`의 `CONFIG.supabase`에 Project URL과 publishable 키가 입력되어 있음
- 테이블 `ahp_responses`와 뷰 4개 생성 완료 (`supabase.sql` + `supabase_update.sql` 적용 상태)

페이지를 열면 자동으로 연결을 점검합니다. 상단에 아무 띠도 뜨지 않으면 정상, 붉은 띠가 뜨면 6절(문제 해결)을 보세요. `CONFIG.supabase`가 비어 있으면 **테스트 모드**로 동작해 전송 대신 저장될 데이터를 화면에 보여줍니다.

## 4. GitHub Pages 배포

1. GitHub **New repository** → 이름 예: `ahp-survey`, Public → Create
2. **Add file → Upload files** → `index.html` 업로드 (저장소 루트) → Commit changes
3. 저장소 **Settings → Pages** → Source: *Deploy from a branch* → Branch: **main**, 폴더: **/ (root)** → Save
4. 1~2분 뒤 접속 주소: `https://<계정명>.github.io/ahp-survey/index.html`

파일 이름을 `index.html`로 바꿔 올리면 `https://<계정명>.github.io/ahp-survey/` 처럼 짧은 주소로 열립니다. 파일 내용은 그대로 두고 이름만 바꾸면 됩니다.

수정본을 올릴 때는 같은 이름으로 다시 업로드(덮어쓰기)하면 1~2분 뒤 반영됩니다. 브라우저 캐시 때문에 바로 안 보이면 새로고침(Ctrl+F5)하세요.

## 5. 응답 데이터

### 5-1. 저장 구조

데이터가 실제로 저장되는 곳은 `ahp_responses` 하나입니다. `v_`로 시작하는 넷은 뷰(view), 즉 그 테이블을 분석하기 좋은 모양으로 실시간 변환해 보여주는 저장된 조회식입니다. 테이블에 응답이 쌓이면 뷰에도 즉시 반영되고, 테이블에서 행을 지우면 뷰에서도 사라집니다. 뷰는 읽기 전용입니다.

**`ahp_responses`** — 응답자 1명 = 1행

| 열 | 내용 |
|---|---|
| `submission_id` | 응답 고유 ID. 같은 ID 재전송은 중복 저장되지 않음 |
| `created_at` / `submitted_at` | 서버 기록 시각 / 브라우저 제출 시각 |
| `affiliation`, `name` | 소속, 이름 |
| `duration_sec` | 설문 시작부터 제출까지 초 |
| `scale` | 사용한 척도 (`1,3,5,7,9`) |
| `cr` | 단계별 CR (JSON) |
| `weights` | 가중치 (JSON 배열) → `v_weights`로 펼쳐 봄 |
| `judgments` | 쌍대비교 값 (JSON 배열) → `v_judgments`로 펼쳐 봄 |
| `answers` | 버튼 선택 원본. 재계산·검증용 보관 |

**뷰 4개**

| 뷰 | 한 행 | 주요 열 | 쓰임 |
|---|---|---|---|
| `v_weights` | 응답자 × 지표 | `level`(area/item), `area`, `item`, `local_w`(영역 내), `global_w`(종합), `cr` | 개별 가중치. 가장 많이 쓰는 뷰 |
| `v_judgments` | 응답자 × 비교쌍 | `left_item`, `right_item`, `value`(a_ij), `choice`(L5·R3 등 원선택) | 쌍대비교 원자료. AIP 재계산용 |
| `v_cr` | 응답자 × 단계 | `section_id`, `cr` | 일관성 점검, 사후 필터링 |
| `v_group_weights` | 지표 (전체 집계) | `n_respondents`, `group_local`, `group_global` | 개별 가중치의 기하평균(AIJ)으로 구한 **집단 가중치** |

- `v_weights`의 `level = 'area'`가 평가영역, `'item'`이 세부지표. 세부지표 `global_w`는 계층(A/B)별 합 1.
- `v_group_weights.group_global`도 계층·수준별 합 1로 정규화.
- `section_id`: A0 안정성 영역, A1 공급, A2 수요, A3 시스템, B0 대응력 영역, B1 상시, B2 비상, B3 인프라.

### 5-2. 내려받기

Supabase **SQL Editor**에서 실행 → 결과 창 **Download CSV**.

```sql
select * from v_weights;         -- 응답자별 가중치
select * from v_judgments;       -- 쌍대비교 원자료
select * from v_cr;              -- 응답자별·단계별 CR
select * from v_group_weights;   -- 집단 가중치 (보고서용)

select * from v_weights where cr <= 0.1;             -- CR 0.1 이하 응답만
select submission_id, affiliation, name, submitted_at
  from ahp_responses order by submitted_at;          -- 응답 현황
```

- 테스트 응답 제거: **Table Editor → ahp_responses**에서 해당 행 삭제, 또는 `supabase_reset.sql` 실행. 뷰에도 즉시 반영.
- `v_group_weights.n_respondents`가 지표마다 다르면 옛 지표 구성으로 제출된 응답이 섞인 것. `jsonb_array_length(weights)`가 25가 아닌 행을 지우면 됨 (reset 파일에 예시).
- AIP 방식(응답자별 쌍대비교 값을 기하평균해 하나의 행렬로 만든 뒤 고유벡터 산출)은 `v_judgments`를 R/Python으로 내려받아 계산.

### 5-3. 무료 플랜 주의

무료 프로젝트는 **일주일간 요청이 없으면 일시정지**됩니다. 설문 기간 중 응답이 뜸한 주에는 대시보드를 한 번 열어 두세요. 일시정지되면 대시보드의 **Restore**로 바로 복구되고 데이터는 유지됩니다.

## 6. 문제 해결

페이지를 열 때와 제출할 때 오류 원인을 화면에 표시합니다.

| 메시지 | 원인 | 조치 |
|---|---|---|
| `Failed to fetch` | URL에 `https://` 누락, 또는 supabase.co 접속 차단 | 현재 파일은 `https://` 누락을 자동 보정함. 그래도 나오면 네트워크 문제 |
| `Invalid API key` | 키가 이 프로젝트 것이 아니거나 재발급됨 | 대시보드 **Connect** 창에서 URL과 키를 함께 복사해 둘 다 교체 |
| `Could not find the 'name' column` | `supabase_update.sql` 미실행 | SQL Editor에서 실행. 반복되면 `notify pgrst, 'reload schema';` 추가 실행 |
| `permission denied for table` (제출 시) | 삽입 권한 정책 누락 | SQL Editor에서 `grant insert on public.ahp_responses to anon;` |
| `Could not find the table` | 다른 프로젝트에 SQL을 실행했거나 미실행 | Table Editor에 `ahp_responses`가 있는 프로젝트의 URL·키를 사용 |
| 붉은 띠 "연결 불가" | 기관 네트워크가 supabase.co 차단 | 휴대폰 테더링으로 확인. 외부 응답자는 영향 없음 |

- 기관 보안장비가 `apikey` 헤더를 제거하는 네트워크에서는 키를 주소에 붙이는 방식으로 자동 전환해 전송합니다.
- 제출 실패 시 응답자는 **응답 파일(JSON)을 내려받아** 담당자에게 보낼 수 있습니다. 담당자는 그 파일 내용을 SQL Editor로 직접 넣을 수 있습니다.
- `/rest/v1/` 루트 주소는 publishable 키로 열리지 않습니다(`Secret API key required`). 정상이며, 점검은 테이블 주소로 합니다.

## 7. 보안

GitHub Pages에 올리면 페이지 소스가 공개되므로 Project URL과 publishable 키도 공개됩니다. 이 둘은 **원래 공개용**이며, 보안은 키가 아니라 데이터베이스 권한 정책이 담당합니다.

- publishable(익명) 키로 가능한 일: `ahp_responses`에 **행 추가**뿐. 읽기·수정·삭제 불가, 뷰 조회 불가. 다른 응답자의 이름·소속·응답을 읽어 갈 수 없음.
- 남는 위험: 링크를 알게 된 제3자가 가짜 응답을 넣는 것. 전문가 수십 명 규모에서는 설문 링크를 참여자에게만 보내는 것으로 관리. (전문가별 초대 토큰 방식은 검토 후 적용하지 않기로 함)
- **절대 올리면 안 되는 것**: `sb_secret_` 키, `service_role` 키, 데이터베이스 비밀번호. 권한 정책을 무시하므로 노출 시 전체 데이터를 읽고 지울 수 있음. 실수로 올렸다면 대시보드에서 즉시 재발급.
- 응답 데이터를 읽을 수 있는 유일한 경로는 Supabase 대시보드 로그인이므로, **Supabase 계정에 2단계 인증**을 켜 두는 것을 권장.
- 응답에 이름·소속이 포함되므로 개인정보 안내 문구를 설문 안내 메일에 넣는 것을 권장.

## 8. 설문 내용 수정

모두 `index.html` 상단 `CONFIG` 블록에서 바꿉니다. 지표를 바꾸면 비교 단계·결과표·전송 데이터가 자동으로 따라옵니다.

```js
crLimit: 0.2,     // 오류 기준 (초과 시 진행 불가)
crWarn:  0.1,     // 주의 기준 (진행은 가능)
showHint: true,   // CR 초과 시 가장 어긋난 쌍을 붉게 표시할지

scale: { values: [1, 3, 5, 7, 9], labels: { 1:"동등하게 중요", … } },

trees: [
  { id: "A", name: "안정성", desc: "…",
    areas: [
      { name: "공급", items: ["…", "…"] },
      …
    ] },
  { id: "B", name: "대응력", … }
]
```

- 항목이 10개를 넘으면 RI표를 벗어나므로 계층을 나누는 편이 좋습니다.
- 글꼴은 `<head>`의 `font-kopubworld` 링크로 불러옵니다. 바탕체로 바꾸려면 `dotum.min.css`를 `batang.min.css`로, CSS의 `'KoPubWorld Dotum'`을 `'KoPubWorld Batang'`으로 바꾸면 됩니다.
- 색상은 CSS `:root`의 변수에서 바꿉니다. 결과표 막대는 `.bar`(평가영역)와 `.bar.g`(세부지표).

## 9. 새 프로젝트를 처음부터 만들 때

1. supabase.com → New project (Region: Northeast Asia) 
2. SQL Editor에 `supabase.sql` 전체 붙여넣고 Run (`supabase_update.sql`은 불필요)
3. 대시보드 **Connect** 창에서 Project URL과 publishable 키 복사 → `index.html`의 `CONFIG.supabase`에 입력
4. 로컬에서 `index.html` 열어 응답 1건 제출 → Table Editor에서 확인 → GitHub Pages에 업로드
