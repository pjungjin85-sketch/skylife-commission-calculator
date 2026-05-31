# skylife-commission-calculator 대화 로그

---

## 세션 7 — 2026-05-31 — 정책 날짜 기반 자동 전환 기능 추가

### 작업 내용

**1. commission.json 구조 변경**
- 기존: `{ "PLANS": {...}, "BASE_FEE": {...}, ... }` (단일 정책 객체)
- 변경: `{ "YYYY-MM-DD": { "PLANS": {...}, ... } }` 날짜 키 구조
- 현재 정책 → `"2026-04-01"` 키로 이관

**2. skylife-plans와의 차이: 날짜 키 형식**
- skylife-plans: `"YYYY-MM"` (매월 1일 고정)
- 수수료 계산기: `"YYYY-MM-DD"` (1일·15일 등 임의 날짜 지원)

**3. index.html — 날짜 기반 자동 분기 로직 추가**
- `DOMContentLoaded` fetch에서 오늘 날짜(`YYYY-MM-DD`) 기준으로 이하 최근 키 선택
- 전역 변수: `ACTIVE_POLICY_KEY`, `POLICY_LABEL` 추가
- 미래 키는 날짜 전까지 자동 무시

**4. 하드코딩 정책 라벨 4곳 제거**
| 위치 | 변경 전 | 변경 후 |
|------|---------|---------|
| 잠금화면 푸터 | `"2026 · 04월 영업정책 · 내부용"` | `id="lockFooterLabel"` JS 자동 생성 |
| 공지 제목 | `"2026년 04월 정책 안내"` | `id="noticeTitleLabel"` JS 자동 생성 |
| 공지 본문 | `"2026년 4월 정책으로 업데이트..."` | `id="noticeBodyLabel"` JS 자동 생성 |
| 헤더 배지 | `"2026 · 04월 영업정책"` | `id="headerBadgeLabel"` JS 자동 생성 |

**5. 공지 팝업 로직 변경**
- 기존: 매일 1회 표시 (`skylife_notice_date`)
- 변경: 정책 키 전환 시 1회만 표시 (`skylife_notice_YYYY-MM-DD`)

**6. showNoticeIfNeeded 호출 타이밍 수정**
- 기존: 스크립트 실행 즉시 (데이터 로드 전, 레이스컨디션)
- 변경: fetch 완료 후 호출 (이미 인증된 경우 DOMContentLoaded에서 처리)

**7. 보안: innerHTML → DOM 메서드**
- `lockFooterLabel` 렌더링: `innerHTML` 대신 `createTextNode` + `createElement('br')` 사용

### 운영 규칙
- 정책 추가: `commission.json`에 `"YYYY-MM-DD": {...}` 키 추가 후 언제든 push
- 예시: 5월 15일 변경 예약 → `"2026-05-15": { ...새 정책... }` 추가
- 이전 키 삭제: 2개 이상 지난 키는 삭제 가능 (직전 1개 유지 권장)

### 커밋
- `3530c41` — feat: 정책 날짜 기반 자동 전환 기능 추가

---

## 세션 1 — 2026-03-27

### 작업 내용: 반응형 UI 구현

**요청**: PC와 모바일에서 화면 사이즈에 맞게 다르게 반응하도록 반응형 UI 수정

---

### 파악한 기존 구조

- `.main-grid`: `grid-template-columns: 1fr 300px` (PC 2열), `@media(max-width:800px)` 1열
- `.top-steps`: `repeat(3, 1fr)`, `@media(max-width:700px)` 1열
- `.result-panel`: `position: sticky; top: 80px` (PC 사이드바)
- `.container`: `max-width: 1100px`
- `.plan-grid`: `auto-fill, minmax(180px, 1fr)`
- 헤더: flex, 우측에 수수료 미리보기 박스(`.header-right`)
- JS `updateResult()`: `hr-amount`(헤더 수수료), `hr-grade` 업데이트

---

### 적용한 변경 사항 (index.html)

#### 1. 태블릿 breakpoint 추가 (≤960px)
```css
@media(max-width:960px) {
  .top-steps { grid-template-columns: repeat(2, 1fr); }
  .top-steps .step-card:last-child { grid-column: span 2; }
}
```
- 상단 3단계 카드를 2열로 재배치
- 3번째 카드는 전체 너비 사용

#### 2. 모바일 CSS (≤640px)
```css
@media(max-width:640px) {
  .header { padding: 10px 14px; gap: 8px; }
  .header-right { display: none; }          /* 헤더 우측 수수료 박스 숨김 */
  .header h1 { font-size: 15px; }
  #mobile-fee-bar { display: flex; }        /* 하단 고정 바 노출 */
  .container { padding: 12px 10px 80px; }  /* 하단 바 높이만큼 여백 */
  .top-steps { grid-template-columns: 1fr; }
  .top-steps .step-card:last-child { grid-column: span 1; }
  .opt-btn { min-width: 80px; padding: 8px 10px; }
  .plan-grid { grid-template-columns: repeat(auto-fill, minmax(140px, 1fr)); }
  .result-panel { position: static; }      /* 모바일에서 sticky 해제 */
}
```

#### 3. 모바일 하단 고정 수수료 바 추가
```html
<div id="mobile-fee-bar">
  <div class="mfb-left">
    <div class="mfb-label">건당 예상 수수료</div>
    <div class="mfb-amount empty" id="mfb-amount">조건 선택 중…</div>
    <div class="mfb-grade" id="mfb-grade"></div>
  </div>
</div>
```
- `position: fixed; bottom: 0` — 화면 하단 고정
- PC에서는 `display: none`, 모바일(≤640px)에서만 노출

#### 4. JS updateResult() / resetAll() 모바일 바 연동
```js
// updateResult 내 — 수수료 계산 완료 시
mfbAmount.className='mfb-amount';
mfbAmount.innerHTML=`${total.toLocaleString()}<span>원</span>`;
mfbGrade.textContent = gradeFee>0 ? `그레이드 +${gradeFee.toLocaleString()}원 포함` : '';

// resetAll 내 — 초기화 시
mfb.className='mfb-amount empty'; mfb.textContent='조건 선택 중…';
document.getElementById('mfb-grade').textContent='';
```

---

### 커밋 & 푸시

- 커밋: `d2d26b3` — 반응형 UI 추가: 모바일/태블릿 레이아웃 최적화
- 브랜치: `main` → `origin/main` push 완료
- 변경: `index.html` +80 lines

---

## 세션 3 — 2026-04-02

### 작업 내용: 4월 정책 업데이트

**요청**: 3개 PDF(2026년 04월 무선 영업정책) 기준으로 4월 정책 내용 반영

---

### PDF 분석 결과

**변경 주요사항 ('26.04월~)**
- 요금제 판매가 변경 및 슬림 1.5GB 판매 중단
- 3개월 이상 미영업 판매점 판매점코드 삭제

**현재 코드 vs 4월 PDF 비교**
| 항목 | 결과 |
|------|------|
| 수수료 체계 (BASE_FEE) | 동일 — 변경 없음 |
| 배정 수수료 | 동일 — 변경 없음 |
| 그레이드 정책 | 동일 — 변경 없음 |
| SKY 현장전용 수수료 | 동일 — 변경 없음 |
| 슬림 1.5GB 판매 중단 | 이미 반영 (코드에 없음) |
| 모두 충분 10GB+(RM) 분류 | 불일치 → 기본형에서 10GB+ 그룹으로 이동 |
| SOS 시리즈 4종 | 누락 → 기본형에 추가 |
| 공지 팝업 내용 | 미입력 → 4월 안내 문구로 업데이트 |

---

### 적용한 변경 사항

#### 1. 기본형 → 10GB+ 그룹으로 이동
- `모두 충분 10GB+` (17,900원 RM): 기본형에서 제거 → 10GB+ 그룹 첫 항목으로 이동
- 이유: 4월 PDF에서 명확히 10GB+ 상품군으로 분류 (수수료 MNP 20,000원 적용)

#### 2. 기본형에 SOS 요금제 4종 추가
```js
["SOS 스쿨 2GB+",    11100, "RM", false, null, null, null, true],
["SOS 스쿨 4GB+",    13100, "RM", false, null, null, null, true],
["SOS 안심 골드 2GB+", 9600, "RM", false, null, null, null, true],
["SOS 안심 골드 4GB+",12100, "RM", false, null, null, null, true],
```

#### 3. 공지 팝업 문구 변경
- 변경 전: "중요 안내 사항 입력" (비어있음)
- 변경 후: "2026년 4월 정책으로 업데이트 되었습니다."

#### 4. 헤더 "← 개통 가이드" 버튼 제거
- 사용자 요청: 기능 없애기
- `<a href="...skylife-guide...">← 개통 가이드</a>` 및 감싸는 `<div>` 제거

---

### 커밋 & 푸시

- 커밋 `b3c48f6` — feat: 4월 정책 업데이트 (SOS 요금제 추가, 10GB+ 재분류, 공지 팝업)
- 커밋 `9118449` — fix: 헤더 개통 가이드 버튼 제거
- 브랜치: `main` → `origin/main` push 완료

---

## 세션 4 — 2026-04-05

### 작업 내용 1: 공지 팝업 닫기 방식 변경

- 기존: `.notice-box` 클릭 시에만 닫힘
- 변경: `#notice-overlay`(딤 배경)에 `onclick="closeNotice()"` 이동, `.notice-box`에 `event.stopPropagation()` 추가 → 아무 곳 클릭해도 닫힘
- 안내 문구: "팝업을 클릭하면 닫힙니다" → "아무 곳이나 클릭하면 닫힙니다"
- 커밋: `b6d84dd`

### 작업 내용 2: MOBILE 배지 글자색 수정

- 원인: `.logo-text span { color: var(--red) }` 규칙이 `.logo-mobile-badge`의 `color: #fff`보다 CSS 특이성이 높아 덮어쓰는 문제
- 수정: `.logo-text span` → `.logo-text span:not(.logo-mobile-badge)` 로 변경
- 커밋: `8d6cef0`

---

## 세션 5 — 2026-04-06

### 작업 내용: 수수료 합계 하단 참고용 주의문구 추가

**요청**: 헤더 우측 수수료 박스와 결과 패널 합계 금액 하단에 주의 문구 삽입

### 적용한 변경 사항

#### 1. 주의문구 CSS 추가
```css
.fee-notice { font-size:9px; color:var(--text-muted); opacity:.7; margin-top:6px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
```
- 9px 작은 글자, 흐린 회색, `white-space:nowrap`으로 한 줄 표시

#### 2. 헤더 우측 결과창 (`.header-result`) 하단 추가
```html
<div class="fee-notice">*해당 금액은 참고용으로 정확한 수수료는 정책서를 참고해주세요</div>
```
- `.hr-grade` 아래 위치
- `min-width` 190px → 300px 로 확장 (문구 잘림 방지)

#### 3. 결과 패널 합계 (`.result-total`) 하단 추가
```html
<div class="fee-notice">*해당 금액은 참고용으로 정확한 수수료는 정책서를 참고해주세요</div>
```
- `#result-total` 바로 아래 위치

### 커밋 & 푸시

- 커밋: `e2ff641` — feat: 수수료 합계 하단 참고용 주의문구 추가
- 브랜치: `main` → `origin/main` push 완료

---

## 세션 6 — 2026-05-21

### 작업 내용: 비밀번호 변경

**요청**: 현행 26년 4월 정책에 맞춰 비밀번호를 `20260401`로 변경

### 적용한 변경 사항

- `index.html` — `CORRECT_HASH` 값 교체
  - 변경 전: `c8fd219f17e06dac4b1977f8a2f343daa1a095b61bccd1ab71db7c98e435ced4`
  - 변경 후: `e20ef9b594a57e2eef2871a1e6e5d53cb8cc74d06afe06b7124b1edc219e5ac0` (`"20260401"` SHA-256)

### 커밋 & 푸시

- 커밋: `1385e74` — 비밀번호 변경: 20260401 (26년 4월 정책 반영)
- 브랜치: `main` → `origin/main` push 완료

---

## 세션 2 — 2026-03-27

### 푸터 디자인 통일

- 기존: `<div>` 태그 + 왼쪽 정렬 (`Created by 박정진`)
- 변경: `<footer>` 태그 + 중앙정렬 + border-top + 페이지명 포함
  - `Created by 박정진 | SKY Life 모바일 수수료 계산기`
- skylife-addons 푸터 스타일 기준으로 전 프로젝트 통일 작업의 일환
- 커밋: `7a73bb3` — style: 푸터 디자인 통일

---
