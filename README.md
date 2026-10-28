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
| `supabase_update_all.sql` | **현재 index.html에 필요한 변경을 한 번에**: 광물별 설문(`mineral` 열, 광물 반영 뷰, `v_progress`, `v_group_weights_wide`) + 응답자 로그인·서버 임시저장(표 3개, 함수 7개, `v_participants`). `supabase.sql`+`supabase_update.sql` 상태에서 실행. 반복 실행 가능, 응답 행 유지 |
| `supabase_update2/3/4.sql` | 위 통합본의 이전 조각들. 통합본을 실행하면 필요 없음 |
| `supabase_reset.sql` | 테스트 데이터 삭제(응답·계정·임시저장·토큰·실패기록). **본 조사 시작 직전에 1회 실행**. 특정 계정·소속·광물만 지우는 선택 삭제 예시 포함. 되돌릴 수 없음 |
| `README.md` | 이 문서 |

## 2. 설문 구성

평가 대상 광물은 **규소 · 마그네슘 · 망간 · 몰리브덴 · 백금** 5종입니다. 광물마다 같은 지표 체계로 설문을 한 번씩 하며, 광물별로 따로 제출되어 `ahp_responses`에 광물당 1행씩 쌓입니다(응답자 1명이 최대 5행).

- **응답자 확인(등록·로그인 통합)**: 폼은 이름·소속·비밀번호 하나뿐입니다. **처음 보는 이름**이면 입력한 비밀번호(**숫자 4자리**)로 등록되고, **이미 있는 이름**이면 비밀번호가 맞을 때 그 계정으로 들어가 **하던 광물의 다음 단계로 바로 이동**합니다(소속은 새로 적은 값으로 갱신). 이름 비교는 띄어쓰기·대소문자를 무시합니다("홍 길동" = "홍길동"). **브라우저를 닫으면 자동 로그아웃**됩니다(같은 탭 새로고침은 유지). 로그인 토큰은 창을 닫으면 사라지는 sessionStorage에만 두고, 그 밖의 어떤 정보도 브라우저에 남기지 않습니다. 서버 토큰은 12시간 미사용 시 만료됩니다. 비밀번호가 틀리면 거절하고 새 계정을 만들지 않으므로 같은 이름으로 계정이 둘로 갈리는 일은 없습니다. 같은 이름으로 15분 안에 10회 틀리면 15분간 차단됩니다(관리자 해제 SQL은 7절). 동명이인은 이름 뒤에 소속을 붙여 구분합니다(예: 홍길동-KEEI). 응답·진행 상태가 모두 서버에 사람별로 저장되므로 브라우저를 닫거나 **다른 기기·다른 브라우저**에서 들어와도 그대로 이어집니다. 같은 브라우저에서는 자동 로그인되고, 로그인하면 맨 위 띠에 "홍길동 님 진행 상황: 망간 응답 중(3/8) · ✓ 규소"와 「이어서 응답」 버튼이 보입니다.
- 로그인 뒤 광물 카드를 눌러 시작합니다. 카드에는 미응답 / 응답 중(n/8단계 완료, 한 단계도 못 끝냈으면 "1단계 진행") / ✓ 제출 완료가 표시됩니다. 같은 사람이 같은 광물을 두 번 제출하는 것은 서버가 막습니다.
- 설문 중에는 화면 상단에 **「현재 평가 광물: 망간 Mn · 자동저장 14:32 · 3 / 5 · 로그아웃」** 띠가 스크롤해도 고정되어 보입니다. 자동저장 시각은 응답을 누를 때마다 갱신되고(저장 중… → 자동저장 hh:mm, 실패 시 "저장 실패"), 로그아웃 버튼으로 어느 단계에서든 나갈 수 있습니다(나가기 전에 저장). 단계 제목·질문 문장·결과표·PDF에도 광물 이름이 붙습니다.
- 제목(상단 h1)을 누르면 안내 화면으로 돌아갑니다. 응답은 그 전에 저장됩니다.
- 제출이 끝나면 남은 광물 목록이 나와 바로 다음 광물로 넘어갈 수 있습니다. 중간에 다른 광물로 옮겨도 각 광물의 응답은 따로 임시저장됩니다.
- 특정 광물만 평가하게 하려면 링크 뒤에 `?m=망간`처럼 붙이면 그 광물만 보입니다(선택 사항).
- 광물 목록은 `CONFIG.minerals`에서 바꿉니다.

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

광물당 30쌍, 약 12분. 5종 전체는 약 1시간이므로 안내 메일에 "나눠서 응답 가능"을 적어 두는 것이 좋습니다.

- **척도**: 5점. Saaty 척도의 다섯 언어적 등급(1 동등 · 3 약간 · 5 더 · 7 매우 · 9 절대적으로)을 씁니다. 1~5 정수 척도로 바꾸는 방법은 `index.html`의 `CONFIG.scale` 주석에 있습니다.
- **일관성 검증**: 항목이 3개 이상인 단계는 응답 완료 즉시 고유벡터법으로 CR을 계산합니다. **CR > 0.2 → 오류 표시, 그 단계는 미완료로 유지**(수정해야 ✓), 0.1 < CR ≤ 0.2 → 주의 표시(완료 처리), 항목 2개인 단계(4·8)는 검증 대상 아님. 오류 시 가장 어긋나는 쌍을 붉은 바탕으로 표시만 하고, 어느 값으로 고치라는 안내는 하지 않습니다.
- **응답자 정보**: 소속과 이름. 둘 다 입력해야 시작됩니다.
- **단계 이동**: 8개 단계는 원하는 순서로 응답할 수 있습니다. 상단의 단계 이름을 누르면 어느 단계로든 이동하고, 완료된 단계(모든 쌍 응답 + CR ≤ 0.2)에는 ✓가 붙습니다. 결과·제출은 모든 단계가 완료되어야 열리며, 제출 후에는 이동이 막힙니다.
- **계층도**: 각 단계 상단에 지표 계층도를 그려 비교 중인 지표를 강조하고, 평가영역 비교 단계(1·5)에는 영역 정의를 함께 표시합니다. 그림은 설정의 지표 이름에서 자동 생성됩니다.
- **결과 화면**: 영역 가중치 × 영역 내 가중치 = 종합 가중치를 표로 보여준 뒤 제출. 계층별 세부지표 종합 가중치 합 = 100%.
- **임시저장**: 응답을 누를 때마다 서버(`ahp_drafts`)에 자동 저장(0.4초 묶음 전송, 단계 이동·탭 전환·창 닫기 때 즉시 전송). 단계 화면 아래에 "망간 응답 저장됨 14:32"가 표시되고, 네트워크 오류 시 "저장 실패"가 뜨며 다음 응답 때 재시도. 제출이 완료되면 임시저장은 삭제.
- **PDF 저장**: 광물 5종을 모두 제출하면 완료 화면과 안내 화면에 「전체 응답 PDF로 저장」 버튼이 나타남. 서버에서 본인 응답을 모두 받아(`ahp_my_responses`) 한 문서로 만들며, 앞에 **요약표(지표 × 광물 5종 종합 가중치)**, 뒤에 광물별 상세(가중치·CR·쌍대비교 30건)가 광물마다 새 페이지로 이어짐. 브라우저 인쇄 대화상자에서 "PDF로 저장"을 고르면 됨. 나중에 다시 로그인해서 받아도 됨. 광물별 개별 PDF는 없음.
- **화면**: 검정 글씨, 각진 모서리, KoPubWorld 돋움(CDN 웹폰트, 미설치 PC에서도 동일). PC·모바일 모두 대응.

## 3. 현재 설정 상태

- Supabase 프로젝트: **KEEI** (`https://ocigorjuluhoylpnthbi.supabase.co`, Northeast Asia/Tokyo, 무료 플랜)
- `index.html`의 `CONFIG.supabase`에 Project URL과 publishable 키가 입력되어 있음
- 테이블 `ahp_responses`와 뷰 생성 완료 (`supabase.sql` + `supabase_update.sql` 적용 상태). **현재 index.html(광물별·로그인 버전)을 쓰려면 `supabase_update_all.sql`을 실행**해야 함. 미실행 시 페이지 상단에 "서버에 로그인 함수가 없습니다" 띠가 뜸

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
| `mineral` | 평가 광물 (규소·마그네슘·망간·몰리브덴·백금) |
| `participant_id` | 응답자 계정 ID (`ahp_participants.id`). 같은 사람·같은 광물은 1행만 허용 |
| `affiliation`, `name` | 소속, 이름 |
| `duration_sec` | 설문 시작부터 제출까지 초 |
| (수집 안 함) | 브라우저·기기 정보(user agent)는 저장하지 않음 |
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
| `v_group_weights` | 광물 × 지표 (집계) | `mineral`, `n_respondents`, `group_local`, `group_global` | **광물별** 집단 가중치 (개별 가중치의 기하평균, AIJ). 세로 형식 |
| `v_group_weights_wide` | 지표 (집계) | `규소`·`마그네슘`·`망간`·`몰리브덴`·`백금` (종합 가중치), `n_*` (응답자 수) | 같은 내용을 **지표 1행 × 광물 5열**로 펼친 보고서용 표. 광물 목록을 바꾸면 SQL의 filter 절도 수정 |
| `v_progress` | 광물 | `n_responses`, `respondents` | 광물별 응답 현황 |
| `v_participants` | 응답자 | `name`, `affiliation`, `last_login`, `submitted`, `in_progress`, `n_submitted` | **응답자별 진행 현황** (누가 어느 광물을 제출했고 어디서 멈췄는지) |

- `v_weights`의 `level = 'area'`가 평가영역, `'item'`이 세부지표. 세부지표 `global_w`는 계층(A/B)별 합 1.
- `v_group_weights.group_global`도 광물·계층·수준별 합 1로 정규화. 모든 뷰에 `mineral` 열이 있으므로 광물별로 걸러서 봅니다.
- `section_id`: A0 안정성 영역, A1 공급, A2 수요, A3 시스템, B0 대응력 영역, B1 상시, B2 비상, B3 인프라.

### 5-2. 내려받기

Supabase **SQL Editor**에서 실행 → 결과 창 **Download CSV**.

```sql
select * from v_weights;         -- 응답자별 가중치
select * from v_judgments;       -- 쌍대비교 원자료
select * from v_cr;              -- 응답자별·단계별 CR
select * from v_group_weights where mineral = '망간';   -- 망간 집단 가중치 (세로)
select * from v_group_weights_wide where level = 'item'; -- 세부지표 × 광물 5종 종합 가중치 (가로, 보고서용)
select * from v_progress;                               -- 광물별 응답 현황
select * from v_participants;                           -- 응답자별 진행 현황 (독촉 대상 확인)

select * from v_weights where cr <= 0.1;             -- CR 0.1 이하 응답만
select submission_id, affiliation, name, submitted_at
  from ahp_responses order by submitted_at;          -- 응답 현황
```

- 테스트 응답 제거: **Table Editor → ahp_responses**에서 해당 행 삭제, 또는 `supabase_reset.sql` 실행. 뷰에도 즉시 반영.
- `v_group_weights.n_respondents`가 지표마다 다르면 옛 지표 구성으로 제출된 응답이 섞인 것. `jsonb_array_length(weights)`가 25가 아닌 행을 지우면 됨 (reset 파일에 예시).
- 테스트 계정 정리: `supabase_reset.sql`의 선택 삭제 B(소속에 '테스트'가 들어간 계정과 그 응답) 또는 A(id 지정). 계정을 지우면 임시저장·토큰은 함께 삭제되지만 제출된 응답은 남으므로 응답을 먼저 지워야 함(파일에 순서대로 적혀 있음).
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
| `Could not find the 'mineral' column` | `supabase_update_all.sql` 미실행 | SQL Editor에서 실행 |
| 상단 띠 "서버에 로그인 함수가 없습니다" / `Could not find the function` | `supabase_update_all.sql` 미실행 | SQL Editor에서 실행 |
| "이 이름은 이미 등록되어 있는데 비밀번호가 다릅니다" | 비밀번호 오타, 또는 동명이인 | 등록 때 비밀번호 입력. 동명이인은 이름 뒤에 소속을 붙임. 분실은 관리자가 재설정(7절) |
| "처음 접속이시군요. 소속을 입력해 주세요" | 새 이름인데 소속 비움 | 소속 입력 |
| "비밀번호는 숫자 4자리로 입력해 주세요" | 형식 오류 | 숫자 4자리 |
| "비밀번호를 여러 번 틀려 잠시 막혔습니다" | 15분 내 10회 실패 | 15분 대기 또는 관리자 해제(7절) |
| 응답자가 "이전 응답이 사라졌다"고 함 | 계정이 둘로 갈림(구버전) | `select * from v_participants;`로 같은 이름 두 줄 확인 → `select ahp_merge_participants(KEEP_ID, DROP_ID);` |
| "접속이 만료되었습니다" | 세션 토큰 삭제됨(관리자가 계정 삭제 등) | 다시 로그인 |
| `permission denied for table` (제출 시) | 삽입 권한 정책 누락 | SQL Editor에서 `grant insert on public.ahp_responses to anon;` |
| `Could not find the table` | 다른 프로젝트에 SQL을 실행했거나 미실행 | Table Editor에 `ahp_responses`가 있는 프로젝트의 URL·키를 사용 |
| 붉은 띠 "연결 불가" | 기관 네트워크가 supabase.co 차단 | 휴대폰 테더링으로 확인. 외부 응답자는 영향 없음 |

- 기관 보안장비가 `apikey` 헤더를 제거하는 네트워크에서는 키를 주소에 붙이는 방식으로 자동 전환해 전송합니다.
- 제출 실패 시 응답자는 **응답 파일(JSON)을 내려받아** 담당자에게 보낼 수 있습니다. 담당자는 그 파일 내용을 SQL Editor로 직접 넣을 수 있습니다.
- `/rest/v1/` 루트 주소는 publishable 키로 열리지 않습니다(`Secret API key required`). 정상이며, 점검은 테이블 주소로 합니다.

## 7. 보안

GitHub Pages에 올리면 페이지 소스가 공개되므로 Project URL과 publishable 키도 공개됩니다. 이 둘은 **원래 공개용**이며, 보안은 키가 아니라 데이터베이스 권한 정책이 담당합니다.

- publishable(익명) 키로 가능한 일: `ahp_*` 함수 7개 호출뿐. 어떤 표도 직접 읽거나 쓸 수 없고, 제출도 `ahp_submit` 함수를 통해 로그인한 계정으로만 가능. 다른 응답자의 이름·소속·응답·비밀번호를 읽어 갈 수 없음.
- 비밀번호(숫자 4자리)는 bcrypt 해시로만 저장되어 관리자도 볼 수 없음. 잊은 전문가가 있으면 SQL Editor에서 재설정: `update ahp_participants set pw_hash = crypt('1234', gen_salt('bf')) where name = '홍길동' and affiliation = '○○대학교';`
- 4자리 숫자는 경우의 수가 1만 개이므로, 틀릴 때마다 0.5초 지연 + 같은 이름 15분 내 10회 실패 시 15분 차단으로 무차별 대입을 막음. 차단 해제: `delete from ahp_login_fails where name_key = regexp_replace(lower('홍길동'), '\s+', '', 'g');` 남는 위험은 링크를 알게 된 제3자가 계정을 만들어 응답하는 것이며, `v_participants`에서 명단에 없는 이름을 걸러낼 수 있음.
- **절대 올리면 안 되는 것**: `sb_secret_` 키, `service_role` 키, 데이터베이스 비밀번호. 권한 정책을 무시하므로 노출 시 전체 데이터를 읽고 지울 수 있음. 실수로 올렸다면 대시보드에서 즉시 재발급.
- 응답 데이터를 읽을 수 있는 유일한 경로는 Supabase 대시보드 로그인이므로, **Supabase 계정에 2단계 인증**을 켜 두는 것을 권장.
- 수집 항목은 이름·소속·응답·제출 시각·소요 시간뿐이며 브라우저·기기 정보는 수집하지 않음. 응답에 이름·소속이 포함되므로 개인정보 안내 문구를 설문 안내 메일에 넣는 것을 권장.

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
