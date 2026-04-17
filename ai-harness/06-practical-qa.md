# [AI 하네스 실전편] 하네스 도입할 때 진짜 궁금한 것들 — 실무 Q&A

&nbsp;

하네스 개념은 알겠다.

규칙 파일, 메모리, Skills, Hooks.

&nbsp;

근데 실제로 도입하려니까 궁금한 게 생긴다.

&nbsp;

"이 규칙 파일을 새 사람한테 어떻게 줘?"

"Skills를 배포할 때 보안 체크는 어떻게 엮어?"

"각 AI 도구마다 경로가 다른데?"

&nbsp;

이 글은 하네스를 실제로 팀에 도입하면서 나온 **실전 질문과 답변**을 정리한 글이다.

&nbsp;

&nbsp;

---

&nbsp;

# Q1. 규칙 파일을 새 사람한테 어떻게 공유해?

&nbsp;

레벨별로 다르다.

&nbsp;

## 프로젝트 레벨 — git에 포함

&nbsp;

```
project/
├── 프로젝트-규칙.md      ← git에 커밋
├── .ai/
│   └── settings.json     ← hooks, 권한도 git에
├── src/
└── package.json
```

&nbsp;

**`git clone`하면 자동으로 받아진다.** 별도 공유 불필요.

새 사람이 할 일: clone → AI 도구 열기 → 끝.

&nbsp;

## 조직 레벨 — 온보딩 스크립트

&nbsp;

```bash
#!/bin/bash
# setup-ai-tools.sh (신규 입사자용)

# 전사 공통 규칙 다운로드
mkdir -p ~/.config/ai-agent
curl -o ~/.config/ai-agent/rules.md \
  https://internal-wiki.company.com/ai-rules.md

echo "AI 도구 설정 완료!"
```

&nbsp;

또는 사내 패키지로:

```bash
npx @company/ai-setup
```

&nbsp;

## 개인 레벨 — 공유 안 함

&nbsp;

개인 선호(응답 스타일, 단축키 등)는 각자 설정.

&nbsp;

## 정리

&nbsp;

```
프로젝트 규칙: git clone하면 끝
조직 규칙:    온보딩 스크립트
개인 설정:    각자 알아서
```

&nbsp;

**가장 중요한 건 프로젝트 규칙이고, 이건 git에 있으니까 자동이다.**

&nbsp;

&nbsp;

---

&nbsp;

# Q2. .ai/ 폴더는 gitignore 대상 아냐?

&nbsp;

맞다. AI 도구의 설정 폴더는 보통 gitignore에 들어간다.

개인 설정(API 키, 로컬 환경)이 들어있으니까.

&nbsp;

하지만 **팀 공유 Skill은 git에 포함해야 한다.**

해결법: gitignore에서 commands/skills 폴더만 예외 처리.

&nbsp;

```gitignore
# AI 도구 설정 (개인)
.ai/

# 팀 공유 Skill은 예외
!.ai/commands/
!.ai/skills/
```

&nbsp;

이러면:

- `settings.json` (개인 설정) → git 제외
- `commands/onboard.md` (팀 Skill) → git 포함

&nbsp;

&nbsp;

---

&nbsp;

# Q3. AI 도구마다 경로가 다른데?

&nbsp;

그렇다. 도구마다 규칙 파일과 Skill의 위치가 다르다.

&nbsp;

| 도구 | 규칙 파일 | Skill/커맨드 |
|:---|:---|:---|
| **Claude Code** | `CLAUDE.md` | `.claude/commands/` |
| **Cursor** | `.cursor/rules/` | `.cursor/rules/` |
| **Windsurf** | `.windsurfrules` | - |
| **GitHub Copilot** | `.github/copilot-instructions.md` | - |

&nbsp;

팀에서 여러 도구를 쓴다면:

&nbsp;

```
project/
├── CLAUDE.md                          # Claude Code
├── .cursor/rules/project.md           # Cursor
├── .github/copilot-instructions.md    # Copilot
└── docs/ai-rules.md                   # 원본 (도구별로 복사)
```

&nbsp;

**원본을 하나 두고 도구별로 복사하는 스크립트**를 만드는 팀도 있다.

```bash
# sync-ai-rules.sh
cp docs/ai-rules.md CLAUDE.md
cp docs/ai-rules.md .cursor/rules/project.md
cp docs/ai-rules.md .github/copilot-instructions.md
```

&nbsp;

현실적으로는 **팀이 쓰는 도구 하나에 맞추는 게** 가장 깔끔하다.

&nbsp;

&nbsp;

---

&nbsp;

# Q4. /onboard Skill은 어떻게 만들어?

&nbsp;

AI 도구의 커맨드 폴더에 마크다운 파일을 만들면 된다.

&nbsp;

```
.ai/commands/onboard.md → /onboard로 실행
```

&nbsp;

```markdown
# /onboard

프로젝트 온보딩을 진행해주세요.

## 1. 프로젝트 개요
- 규칙 파일을 읽고 프로젝트 개요를 설명해주세요
- 기술 스택, 아키텍처를 보여주세요

## 2. 디렉토리 구조
- 각 앱/패키지의 역할을 설명해주세요

## 3. 개발 환경 셋업
- 필요한 도구 버전 확인
- 의존성 설치 방법
- 환경 변수 설정 방법
- 개발 서버 실행 방법

## 4. 주요 파일 위치
- API 클라이언트, 공유 컴포넌트, 상태 관리, 유틸리티

## 5. 자주 사용하는 명령어
- 빌드, 배포, 테스트 명령어 정리

## 6. 팀 컨벤션
- Git 커밋, 배포 규칙, 코딩 스타일 요약

간결하게 요약해서 설명해주세요.
```

&nbsp;

새 팀원이 `/onboard` 입력하면 AI가 프로젝트를 분석해서 설명해준다.

사람이 설명하는 것보다 **빠지는 게 없다.**

&nbsp;

&nbsp;

---

&nbsp;

# Q5. /deploy할 때 보안 체크, 코드 리뷰는 어떻게 엮어?

&nbsp;

`/deploy`만 하면 보안 체크, 코드 리뷰를 건너뛸 수 있다.

두 가지로 방어한다.

&nbsp;

## 방법 1: Skill 안에 전체 흐름 포함

&nbsp;

```markdown
# /deploy

## 배포 전 자동 체크

### 1단계: 보안 점검
- 하드코딩된 비밀번호/토큰 검색
- console.log에 민감정보 출력 확인
- .env 파일이 gitignore에 있는지 확인

### 2단계: 코드 리뷰
- 변경된 파일의 에러 처리 확인
- 미사용 import/변수 확인
- 빌드 에러 확인

### 3단계: 배포
- 변경된 앱만 버전 올리기
- 커밋 + 푸시
- 배포 스크립트 실행
- 결과물 검증

하나라도 실패하면 배포를 중단하고 알려주세요.
```

&nbsp;

## 방법 2: Hooks로 강제

&nbsp;

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(npm run deploy*)",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint && npm run test"
          }
        ]
      }
    ]
  }
}
```

&nbsp;

## 차이

&nbsp;

| | Skill | Hook |
|:---|:---|:---|
| **강제성** | 권고 (건너뛸 수 있음) | 강제 (못 건너뜀) |
| **유연성** | 높음 (AI가 판단) | 낮음 (스크립트 실행) |
| **용도** | 전체 워크플로우 가이드 | 특정 조건 강제 |

&nbsp;

**추천: 둘 다 쓴다.**

Skill에서 전체 흐름(보안 → 리뷰 → 배포)을 정의하고,

Hooks로 빌드/린트는 강제한다.

&nbsp;

Skill은 "이렇게 해주세요"고,

Hook은 "이거 안 하면 실행 안 됩니다"다.

&nbsp;

&nbsp;

---

&nbsp;

# Q6. 규칙 파일에 뭘 얼마나 써야 해?

&nbsp;

## 못 쓴 규칙

&nbsp;

```markdown
# 규칙
- 코드를 깨끗하게 작성하세요
- 좋은 변수명을 사용하세요
- 테스트를 잘 작성하세요
```

&nbsp;

이건 사람한테도 쓸모없고, AI한테도 쓸모없다. **모호하다.**

&nbsp;

## 잘 쓴 규칙

&nbsp;

```markdown
# 빌드
- npm run dev:app-a  # 서비스 A (port 3000)
- npm run dev:app-b  # 서비스 B (port 3001)

# 코딩 스타일
- 함수 30줄 이내
- 파라미터 4개 이내
- any 타입 사용 금지
- console.log 커밋 금지

# 배포
- 변경된 앱만 버전 올리기 (4번째 자리 +1)
- npm run deploy:{app}:{profile} 사용
- 수동 빌드 금지 (env 사고 위험)

# 금지
- main 브랜치 직접 push 금지
- .env 파일 커밋 금지
```

&nbsp;

**구체적이고, 실행 가능하고, AI가 바로 따를 수 있다.**

&nbsp;

## 분량

&nbsp;

- 시작: 50줄
- 안정화: 100~150줄
- 최대: 200줄

&nbsp;

200줄 넘으면 AI가 중요한 규칙을 놓치기 시작한다.

많으면 파일을 분리하고 `@import`로 참조한다.

&nbsp;

&nbsp;

---

&nbsp;

# Q7. 메모리는 어떻게 공유해?

&nbsp;

**공유하지 않는다.**

&nbsp;

메모리는 개인별로 축적되는 거다.

&nbsp;

```
개발자 A의 메모리:
  "배포할 때 항상 staging 스크립트 써줘" → 저장

개발자 B의 메모리:
  (이 규칙 모름)
```

&nbsp;

공유해야 할 정도로 중요한 규칙이면 **메모리가 아니라 규칙 파일에 넣는 게 맞다.**

&nbsp;

```
메모리에 남길 것: 개인 선호, 작업 히스토리
규칙 파일에 넣을 것: 팀 전체가 따라야 하는 규칙
```

&nbsp;

메모리에서 반복적으로 쌓이는 피드백이 있으면, 그건 규칙 파일로 승격시킨다.

&nbsp;

&nbsp;

---

&nbsp;

# Q8. Skill을 팀원끼리 공유하려면?

&nbsp;

gitignore에서 commands 폴더만 예외 처리하면 된다.

&nbsp;

```gitignore
# AI 도구 설정
.ai/

# 팀 공유 Skill은 예외
!.ai/commands/
```

&nbsp;

```
.ai/
├── settings.json        ← git 제외 (개인 설정)
└── commands/
    ├── onboard.md       ← git 포함 (팀 공유)
    ├── deploy.md        ← git 포함
    └── review.md        ← git 포함
```

&nbsp;

clone하면 Skill이 자동으로 받아지고, 누구나 `/onboard`, `/deploy` 실행 가능.

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

하네스 도입의 핵심은 **"특별한 작업 없이 자동으로 적용되는 구조"**다.

&nbsp;

- 규칙 파일: git에 있으니까 clone하면 끝
- Skill: commands 폴더만 git 예외 처리
- Hook: settings.json으로 강제
- 메모리: 개인별 축적, 중요하면 규칙 파일로 승격

&nbsp;

새 사람이 해야 할 일:

```
1. git clone
2. AI 도구 설치
3. /onboard
```

&nbsp;

이 3단계로 프로젝트 이해 + 개발 시작이 가능하면 성공이다.

&nbsp;

&nbsp;

---

AI하네스, Skills, Hooks, 온보딩, 팀공유, gitignore, 프로젝트규칙, AI코딩, 개발생산성, 바이브코딩, 팀도입, 커스텀명령
