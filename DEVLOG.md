# 🍎 사과 게임 — 개발 기록

## 프로젝트 개요

숫자가 적힌 사과를 드래그로 선택해 합이 **10**이 되면 사과가 사라지는 퍼즐 게임.  
기존 단일 HTML 게임 파일에 Supabase + Vercel을 연동해 로그인·닉네임·리더보드 기능을 추가했다.

---

## 기술 스택

| 항목 | 내용 |
|------|------|
| 프론트엔드 | 단일 HTML 파일 (Vanilla JS) |
| 인증 / DB | Supabase (Google OAuth, PostgreSQL) |
| 배포 | Vercel (GitHub 자동 배포) |
| Supabase JS SDK | CDN (`@supabase/supabase-js@2`) |

---

## 구현 기능

### 1. Google 로그인
- Supabase OAuth를 통한 Google 소셜 로그인
- 로그인 상태는 페이지 새로고침 후에도 세션 유지
- 로그아웃 버튼 — 시작 화면에 배치

### 2. 닉네임 설정 / 변경
- 최초 로그인 시 닉네임 설정 화면 자동 표시
- 조건: 한글·영문·숫자, 2~15자, 중복 불가
- 시작 화면에서 언제든 **✏️ 별명 변경** 가능
- 게임 중 / 게임 전 상태에 따라 저장 후 돌아가는 화면 분기 처리

### 3. 점수 저장 및 리더보드
- 게임 종료 시 Supabase `leaderboard` 테이블에 자동 저장
- 게임 종료 화면에서 TOP 10 순위 표시 (내 점수 강조 ★)
- 시작 화면에서 🏆 순위표 버튼으로 언제든 조회 가능
- 메달 표시: 🥇 🥈 🥉

### 4. 게스트 플레이
- 로그인 없이 **👤 게스트로 플레이** 선택 가능
- 게스트도 닉네임 설정 가능 (메모리에만 저장, 새로고침 시 초기화)
- 점수는 리더보드에 저장되지 않음
- 게임 종료 후 순위표 조회는 가능 + 로그인 유도 메시지 표시
- HUD에 `닉네임 (게스트)` 표시

---

## 화면 흐름

```
앱 실행
  └─ 로그인 화면
        ├─ Google로 로그인
        │     └─ (첫 로그인) 닉네임 설정 → 시작 화면
        │     └─ (재방문)    시작 화면
        └─ 게스트로 플레이
              └─ 닉네임 설정 → 시작 화면

시작 화면
  ├─ 게임 시작 → 인게임
  ├─ 🏆 순위표
  ├─ ✏️ 별명 변경
  └─ 로그아웃 → 로그인 화면

인게임
  └─ 시간 종료 → 게임 종료 화면
        ├─ 점수 저장 (로그인 유저만)
        ├─ TOP 10 순위 표시
        └─ 다시 하기 → 시작 화면
```

---

## Supabase 설정

### 테이블 스키마

**`profiles`** — 유저별 닉네임 저장

```sql
create table public.profiles (
  id         uuid references auth.users on delete cascade primary key,
  nickname   text unique not null,
  created_at timestamp with time zone default timezone('utc', now())
);
```

**`leaderboard`** — 게임 점수 기록

```sql
create table public.leaderboard (
  id         uuid default gen_random_uuid() primary key,
  user_id    uuid references auth.users on delete cascade,
  nickname   text not null,
  score      integer not null check (score >= 0),
  created_at timestamp with time zone default timezone('utc', now())
);
```

### RLS 정책

```sql
-- profiles
create policy "누구나 조회 가능"      on public.profiles for select using (true);
create policy "본인만 등록 가능"      on public.profiles for insert with check (auth.uid() = id);
create policy "본인만 수정 가능"      on public.profiles for update using (auth.uid() = id);

-- leaderboard
create policy "누구나 조회 가능"      on public.leaderboard for select using (true);
create policy "본인 점수만 등록 가능" on public.leaderboard for insert with check (auth.uid() = user_id);
```

### Google OAuth 설정 순서

1. Supabase 대시보드 → **Authentication → Providers → Google** 활성화
2. Google Cloud Console → OAuth 2.0 클라이언트 생성
   - 승인된 리디렉션 URI: `https://<project>.supabase.co/auth/v1/callback`
3. Client ID / Secret을 Supabase에 입력
4. Supabase → **Authentication → URL Configuration**
   - Site URL: 배포된 Vercel URL
   - Redirect URLs: 배포된 Vercel URL

---

## Vercel 배포

### 배포 방법 (GitHub 연동)

1. GitHub 저장소에 `index.html` push
2. [vercel.com](https://vercel.com) → **Add New → Project** → 저장소 Import
3. 설정 변경 없이 **Deploy**
4. 생성된 URL을 Supabase URL Configuration에 등록

### 이후 업데이트

```bash
git add index.html
git commit -m "변경 내용"
git push origin main
# → Vercel 자동 재배포
```

---

## 파일 구조

```
apple bomb/
├── index.html          # 게임 전체 (HTML + CSS + JS)
├── supabase-schema.sql # Supabase SQL 스키마
├── DEVLOG.md           # 이 파일
├── appleimage.png
├── bomb_effect.png
├── bombapple.png
└── bombapple_iced.png
```

---

## index.html 주요 구조

```
<head>
  Supabase JS CDN
  CSS (게임 스타일 + 오버레이 + 리더보드)
</head>
<body>
  HUD (점수 / 타이머 / 닉네임 / 초기화)
  게임 그리드

  [오버레이 5종]
  ├── #auth-overlay       로그인 화면
  ├── #nickname-overlay   닉네임 설정/변경
  ├── #start-overlay      게임 시작 화면
  ├── #leaderboard-overlay 순위표
  └── #end-overlay        게임 종료 화면

  <script>
    Supabase 초기화 (SUPABASE_URL, SUPABASE_ANON_KEY)
    인증 상태 변수 (currentUser, userNickname, isGuest)
    게임 로직 (grid, bombType, 드래그 선택, 폭탄 처리)
    인증 함수 (initAuth, signInWithGoogle, signOut, playAsGuest)
    닉네임 함수 (openNicknameOverlay, saveNickname)
    리더보드 함수 (recordScore, fetchLeaderboard, renderLeaderboardHTML)
  </script>
</body>
```

---

## Git 커밋 히스토리

| 커밋 | 내용 |
|------|------|
| `85cc088` | 초기 게임 버전 (로그인 없음) |
| `01f7ac5` | Supabase 인증 · 닉네임 · 리더보드 추가, 파일명 index.html로 변경 |
| `c90781f` | 닉네임 변경 · 로그아웃 버튼 추가 |
| `db7f845` | 버튼 위치 인게임 HUD → 시작 화면으로 이동 |
| `b03719a` | 게스트 플레이 (닉네임 포함) 추가 |
