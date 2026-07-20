# skylife-commission-calculator 프로젝트 규칙

공통 규칙은 상위 디렉토리 `/Users/jaypark/workspace/CLAUDE.md` 참고.

## 이 페이지 정보
- **용도**: 스카이라이프 모바일 수수료 계산기 (대리점 영업용)
- **파일**: `index.html` (단일 파일)
- **GitHub Pages URL**: https://pjungjin85-sketch.github.io/skylife-commission-calculator/

## 기능 구조
- 요금제 선택 + 슬라이더로 등급 입력
- HR(현물지원금), MFB(마케팅) 수수료 계산 및 결과 표시
- `grade-slider` → `--pct` CSS 변수로 시각적 표현

## 수정 시 주의사항
- 수수료 계산 로직은 실제 정책 기반 — 임의로 수치 변경 금지
- `#grade-result-box` display 토글은 결과 표시 로직과 연결됨
- 슬라이더 CSS `--pct` 변수는 JS에서 직접 set함 (CSS 단독 수정 금지)
