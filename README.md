# 후보 발굴 데스크

아이돌 **공식 유튜브 영상의 자막**에서 팬들이 따라 살 만한 **제품**과 찾아갈 만한 **장소** 후보를 찾아,
관리자가 정확한 영상 시점을 열어 보고 **승인·보류·제외**하는 관리자 전용 웹앱입니다.

- 관리자는 아이돌 **이름만** 등록합니다. 채널 주소를 넣지 않아도 공식 채널을 스스로 찾고, 확신하지 못한 출처만 "출처 확인"으로 보냅니다.
- 과거 영상(기본 최근 180일)과 신규 영상(기본 하루 2회 확인)을 예약 실행으로 처리합니다.
- 후보마다 **자막 원문 근거와 시작 초**가 붙고, 근거를 자막에서 찾지 못한 후보는 "검증 실패"로 따로 둡니다.
- 승인한 후보는 JSON 으로 내려받거나 API 로 가져갈 수 있습니다.

기술: Next.js 16 · Supabase(Postgres, Auth) · Vercel · YouTube Data API v3 · Google Gemini(무료 한도)

---

## 처음 설정하기 (한 번만)

### 1. GitHub 저장소

1. github.com → **New repository** → 이름 `candidate-desk` → **Private** → README 등은 체크하지 않고 **Create**
2. 받은 ZIP 을 풀고, 저장소 첫 화면에서 **uploading an existing file** → 폴더 안의 파일과 폴더를 **전부** 끌어다 놓기
   (`src`, `supabase`, `package.json`, `package-lock.json`, `.npmrc`, `next.config.ts`, `vercel.json` … 이 저장소 최상위에 보여야 합니다)
3. 점으로 시작하는 파일(`.npmrc`, `.gitignore`, `.env.example`)이 안 보이면 맥 Finder 에서 `Cmd + Shift + .` 을 눌러 보이게 한 뒤 함께 올려 주세요.

### 2. Supabase 프로젝트

1. supabase.com/dashboard → **New project** → 이름 `candidate-desk`, Region **Northeast Asia (Seoul)**
2. **SQL Editor → New query** 에 `supabase/migrations/` 의 SQL 파일을 **파일명 순서대로** 하나씩 붙여 넣고 **Run**
   ("Success. No rows returned" 가 나오면 됩니다)
3. **Authentication → Sign In / Providers**
   - Email 켜짐 확인
   - **Allow new users to sign up 끄기**
4. **Authentication → Users → Add user → Create new user** 로 관리자 계정 만들기(이메일·비밀번호, **Auto Confirm User** 체크)
5. **Project Settings → API Keys** 에서 복사해 두기
   - Project URL
   - Publishable key (`sb_publishable_…`)
   - Secret key (`sb_secret_…`) ← 서버 전용, 절대 공유 금지

### 3. YouTube Data API 키 (무료)

1. console.cloud.google.com → 새 프로젝트
2. **API 및 서비스 → 라이브러리 → YouTube Data API v3 → 사용**
3. **사용자 인증 정보 → 사용자 인증 정보 만들기 → API 키**
4. 키 편집 → API 제한사항: **YouTube Data API v3** 만 선택
5. 하루 10,000 단위 무료. 검색 1회 = 100단위, 나머지 조회는 1단위입니다.

### 4. Gemini API 키 (무료)

1. **aistudio.google.com** → 로그인 → 왼쪽 아래 **Get API key → Create API key**
2. 결제 정보를 등록하지 않으면 **무료 한도** 안에서만 동작하고 요금이 나가지 않습니다.
   무료 한도(분당·하루 요청 수)를 넘으면 그 영상은 다음 실행으로 자동으로 미뤄집니다.
3. 다른 모델을 쓰려면 환경 변수 `CANDIDATE_MODEL` 에 AI Studio 에 나오는 모델 ID 를 넣습니다(기본 `gemini-3.6-flash`).

> 무료 한도로 보낸 내용은 Google 이 서비스 개선에 쓸 수 있습니다. 여기서 보내는 것은 공개 영상의 자막뿐입니다.

### 5. Vercel 배포

1. vercel.com → **Add New → Project** → `candidate-desk` 저장소 **Import**
2. **Environment Variables** 에 아래를 넣고 **Deploy**

| 이름 | 값 |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase Project URL |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Publishable key |
| `SUPABASE_SECRET_KEY` | Secret key |
| `ADMIN_EMAILS` | 관리자 이메일(여러 명이면 쉼표로) |
| `YOUTUBE_API_KEY` | 3번에서 만든 키 |
| `GEMINI_API_KEY` | 4번에서 만든 키 |
| `PIPELINE_SECRET` | 아무도 모르는 긴 문자열(40자 이상 권장) |
| `CRON_SECRET` | `PIPELINE_SECRET` 과 같은 값 |
| `EXPORT_API_KEY` | (선택) 외부 시스템용 긴 문자열 |

긴 무작위 문자열은 터미널에서 `openssl rand -hex 24` 로 만들 수 있습니다.

3. 배포가 끝나면 주소(예: `https://candidate-desk.vercel.app`)로 들어가 2-4 에서 만든 계정으로 로그인합니다.

### 6. 예약 실행 켜기

`vercel.json` 에 **하루 1번(한국 시간 오전 6시)** 실행이 이미 들어 있습니다(Vercel 무료 플랜 한도).
더 자주(20분마다) 돌리려면 Supabase **SQL Editor** 에서 아래를 실행하세요. `<배포 주소>` 와 `<PIPELINE_SECRET>` 를 바꿔 넣습니다.

```sql
create extension if not exists pg_cron;
create extension if not exists pg_net;

select cron.schedule(
  'candidate-desk-tick',
  '*/20 * * * *',
  $$
  select net.http_post(
    url := 'https://<배포 주소>/api/tick',
    headers := jsonb_build_object('Authorization', 'Bearer <PIPELINE_SECRET>'),
    timeout_milliseconds := 300000
  );
  $$
);
```

한 번 실행할 때마다: 확인할 때가 된 아이돌을 **오래 안 본 순서로** 90초 동안 돌아가며 새 영상을 찾고(기본 12시간 간격), 대기 중인 영상을 최대 2개 처리합니다.
멈추려면 `select cron.unschedule('candidate-desk-tick');`

---

## 쓰는 법 (왼쪽 메뉴)

- **후보 검수**: 매일 보는 화면. 목록에서 후보를 누르면 오른쪽에 영상(그 시점부터 재생)·자막 맥락·검수 메모가 뜹니다. 승인·보류·제외를 누르고, 여러 개는 체크해서 한꺼번에 처리합니다. "새 후보만"으로 아직 안 본 것만 볼 수 있고, 근거를 못 찾은 후보는 "검증 실패" 버튼에 따로 모입니다.
- **영상**: 찾아낸 영상이 어디까지 처리됐는지 봅니다(미처리·완료·보류·실패·출처 확인 대기·제외). 실패·보류 영상은 체크해서 "지금 처리". 제목을 누르면 자막을 직접 붙여 넣을 수 있습니다.
- **출처 확인**: 시스템이 공식 출처인지 확신하지 못한 채널만 모입니다(구독자 10만 명 이상인데 공식 목록에 없는 채널 등). 공식 채널이면 "공식 출처로 확인"(아티스트 본인 채널이면 종류를 '아티스트'로 두고 아이돌 선택), 아니면 "제외". 한 번 정하면 그 채널의 다음 영상부터는 자동입니다. 작은 팬·정리·쇼핑 채널은 자동으로 제외되며 "판정이 끝난 채널"에서 되돌릴 수 있습니다. 목록이 쌓였으면 "남은 채널 모두 제외".
- **아이돌**: 이름만 넣어 등록 → "지금 확인"으로 영상 찾기(두 번째부터는 지난 확인 이후 새 영상만). "영상 추가"에 주소를 넣으면 그 영상 하나만 바로 처리합니다. "과거 기간 다시 훑기"는 다음 확인 때 최근 N일(기본 180일)을 처음처럼 다시 뒤집니다.
  - 영상 찾는 곳: ① 확인된 아티스트 공식 채널의 업로드 전체(할당량 거의 안 듦) ② "이름 + 브이로그|vlog|왓츠인마이백…" 검색 1번 ③ 자체 콘텐츠(자컨) 이름 검색 ④ 추가 검색어.
  - 제목에 직캠·음악방송·콘서트·MV·안무 등이 있거나 90초 미만이면 처리하지 않고 제외합니다(비하인드·브이로그 등은 살림). 단어 목록은 설정에서 고칩니다.
- **실행 기록·설정**: 자동 실행 기록, 유튜브 할당량, AI 사용량, 확인 간격 등 설정, 승인 후보 JSON 받기.
- **진단**: 영상 주소 하나로 유튜브 정보 → 자막 → AI 연결 → 후보 추출을 차례로 시험해 어디서 막히는지 보여 줍니다(저장하지 않음).

### 자막을 못 가져올 때

유튜브는 서버(Vercel)에서 오는 자막 요청을 "봇 확인"으로 자주 막습니다. 막히면 **Gemini 가 공개 유튜브 영상을 직접 보고 들으며** 후보를 찾습니다(Gemini API 가 공식 지원하는 기능, 20분 단위로 나눠 봄). 이 경우 자막 칸에 "AI 영상 분석(자막 없음)"으로 표시되고, 후보에 "영상에서 시점 확인 필요" 메모가 붙습니다.
자막을 붙여 넣으면(영상 → 제목 → 자막 붙여 넣기) 자막 기준으로 다시 뽑습니다.

### 외부에서 승인 후보 가져가기

```
GET https://<배포 주소>/api/v1/candidates?idol=엔하이픈&since=2026-10-01T00:00:00Z
x-api-key: <EXPORT_API_KEY>
```

각 항목의 앞 7개 필드(`timestamp`, `timestamp_url`, `members`, `type`, `candidate_name`, `context`, `idea_note`)는 과업지시서의 형식 그대로이고, 뒤에 `video_id`, `evidence`, `mention_type`, `review_status` 등이 붙습니다.

---

## 구조

```
src/app/(desk)/        관리자 화면 (후보 검수, 영상, 출처 확인, 아이돌, 실행 기록·설정)
src/app/api/tick       예약 실행 입구
src/app/api/v1         외부용 승인 후보 API
src/app/actions.ts     화면의 모든 쓰기 작업(서버 액션, 관리자 확인 후 실행)
src/lib/pipeline/      youtube.ts(영상·채널) · sources.ts(공식 출처 판정) · transcript.ts(자막)
                       extract.ts(AI 후보 추출) · verify.ts(근거 검증·중복 판정) · run.ts(전체 흐름)
supabase/migrations/   데이터베이스 구조
```

보안: 모든 테이블은 RLS 로 잠겨 있어 브라우저 키로는 아무것도 읽을 수 없고, 서버만 secret 키로 접근합니다.
로그인은 Supabase 계정 + `ADMIN_EMAILS` 에 있는 이메일만 통과합니다.

## 개발

```
npm install
cp .env.example .env.local   # 값 채우기
npm run dev
npm test          # 파이프라인 단위 테스트
npm run typecheck && npm run lint && npm run build
```
