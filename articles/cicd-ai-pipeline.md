# CI/CD에 AI를 끼얹으면 — PR부터 프로덕션 배포까지 자동화하기

&nbsp;

PR 올리면 이런 일이 벌어진다고 상상해보자.

&nbsp;

```
1. AI가 코드 리뷰를 단다
2. 테스트가 자동으로 돌아간다
3. 스테이징에 자동 배포된다
4. 팀장에게 "확인해주세요" 알림이 간다
5. 팀장이 버튼 하나 누르면 프로덕션에 배포된다
```

&nbsp;

개발자가 하는 일: **PR 올리기 + 승인 버튼.**

나머지는 전부 자동이다.

&nbsp;

이 글은 이 파이프라인을 **처음부터 끝까지** 만드는 가이드다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. 전체 흐름

&nbsp;

```
개발자: git push → PR 생성
  ↓ 자동
CI: lint → build → test
  ↓ 전부 통과하면
AI: 코드 리뷰 코멘트 (자동)
  ↓ 리뷰 반영
스테이징: 자동 배포
  ↓
팀장에게 알림: "스테이징에서 확인해주세요"
  ↓ 직접 확인
팀장: GitHub에서 [Approve] 클릭
  ↓
프로덕션: 자동 배포 + 헬스체크 + Slack 알림
```

&nbsp;

각 단계를 하나씩 만들어보자.

&nbsp;

&nbsp;

---

&nbsp;

# 2. Step 1 — 테스트 자동 실행

&nbsp;

PR이 올라오면 GitHub Actions가 자동으로 돌아간다.

&nbsp;

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Node.js 설치
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: 의존성 설치
        run: npm ci

      - name: 린트
        run: npm run lint

      - name: 빌드
        run: npm run build

      - name: 테스트
        run: npm test
```

&nbsp;

```
개발자가 PR 올림
  → GitHub가 자동 감지
  → 서버(runner)에서 코드 받아서
  → npm ci → lint → build → test 실행
  → 결과를 PR에 표시 (✅ 통과 / ❌ 실패)
```

&nbsp;

## "테스트가 없는데요?"

&nbsp;

현실적으로 테스트 코드가 없는 프로젝트가 많다.

그래도 lint + build만으로 가치가 있다.

&nbsp;

| 테스트 수준 | 확인하는 것 | 가치 |
|:---|:---|:---|
| lint만 | 문법 에러, 미사용 변수 | "코드가 깨끗하다" |
| lint + build | 타입 에러, import 누락 | "빌드가 된다" |
| lint + build + test | 기능 동작 | "기능이 맞다" |

&nbsp;

**테스트 없어도 lint + build만으로 "머지하면 깨지는 PR"을 막을 수 있다.**

&nbsp;

&nbsp;

---

&nbsp;

# 3. Step 2 — AI 코드 리뷰

&nbsp;

테스트 통과하면 AI가 자동으로 코드 리뷰 코멘트를 단다.

&nbsp;

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    branches: [main]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    needs: test  # 테스트 통과 후에만
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: 변경된 파일 가져오기
        id: diff
        run: |
          DIFF=$(git diff origin/main..HEAD -- '*.ts' '*.tsx' '*.js')
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$DIFF" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: AI 리뷰 요청
        uses: actions/github-script@v7
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          script: |
            const diff = `${{ steps.diff.outputs.diff }}`;
            
            const response = await fetch('https://api.anthropic.com/v1/messages', {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json',
                'x-api-key': process.env.ANTHROPIC_API_KEY,
                'anthropic-version': '2023-06-01'
              },
              body: JSON.stringify({
                model: 'claude-sonnet-4-20250514',
                max_tokens: 2000,
                messages: [{
                  role: 'user',
                  content: `다음 코드 변경사항을 리뷰해줘. 
                    버그, 보안 취약점, 성능 문제만 지적해줘.
                    사소한 스타일 지적은 하지 마.
                    문제 없으면 "LGTM"이라고만 해줘.\n\n${diff}`
                }]
              })
            });
            
            const data = await response.json();
            const review = data.content[0].text;
            
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🤖 AI Code Review\n\n${review}`
            });
```

&nbsp;

PR에 이런 코멘트가 자동으로 달린다:

```
## 🤖 AI Code Review

1. src/api/users.ts:42 — user가 null일 수 있는데 체크 없이 접근하고 있습니다.
   ```
   const name = user.name;  // user가 null이면 TypeError
   → const name = user?.name || 'Unknown';
   ```

2. src/auth/login.ts:15 — 비밀번호를 평문으로 비교하고 있습니다.
   보안 위험: bcrypt.compare()를 사용하세요.

나머지는 LGTM 👍
```

&nbsp;

**사람 리뷰 전에 기본적인 문제를 AI가 걸러준다.**

&nbsp;

&nbsp;

---

&nbsp;

# 4. Step 3 — 스테이징 자동 배포

&nbsp;

테스트 통과 + AI 리뷰 완료되면 스테이징에 자동 배포한다.

&nbsp;

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize]

jobs:
  # Step 1, 2는 위에서 이미 정의

  deploy-staging:
    runs-on: ubuntu-latest
    needs: [test, ai-review]
    steps:
      - uses: actions/checkout@v4

      - name: Docker 이미지 빌드
        run: docker build -t myapp:${{ github.sha }} .

      - name: 레지스트리에 푸시
        run: |
          echo ${{ secrets.REGISTRY_PASSWORD }} | docker login registry.company.com -u deploy --password-stdin
          docker tag myapp:${{ github.sha }} registry.company.com/myapp:staging
          docker push registry.company.com/myapp:staging

      - name: 스테이징 서버에 배포
        run: |
          ssh deploy@staging.company.com "
            docker pull registry.company.com/myapp:staging
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp -p 3000:3000 registry.company.com/myapp:staging
          "

      - name: 헬스체크
        run: |
          sleep 10
          curl -f https://staging.company.com/health || exit 1

      - name: Slack 알림
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "🚀 스테이징 배포 완료\nPR: #${{ github.event.number }}\n확인: https://staging.company.com\n\n승인: ${{ github.event.pull_request.html_url }}"
            }'
```

&nbsp;

```
테스트 + AI 리뷰 통과
  → Docker 이미지 빌드
  → 스테이징 서버에 자동 배포
  → 헬스체크 (정상 확인)
  → Slack: "스테이징 배포 완료, 확인해주세요"
```

&nbsp;

&nbsp;

---

&nbsp;

# 5. Step 4 — 사람이 확인 + 승인

&nbsp;

이 단계가 가장 궁금한 부분이다.

**"사람이 확인"이라는 게 대체 어떻게 동작하는 거야?**

&nbsp;

GitHub에 **Environment Protection Rules**가 있다.

&nbsp;

## 설정 방법

&nbsp;

```
GitHub → Settings → Environments → New environment
  이름: production
  Required reviewers: @team-lead, @senior-dev
  Wait timer: 0 (바로 승인 가능)
```

&nbsp;

## 워크플로우에 적용

&nbsp;

```yaml
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production        # ← 이 한 줄이 핵심
    steps:
      - run: echo "프로덕션 배포 시작"
      # ... 배포 스크립트
```

&nbsp;

`environment: production`을 추가하면, 이 job이 실행되기 전에 **지정된 사람의 승인을 기다린다.**

&nbsp;

## 실제 화면

&nbsp;

```
┌─────────────────────────────────────────┐
│ ⏳ Waiting for review                    │
│                                         │
│ deploy-production                       │
│ Environment: production                 │
│                                         │
│ Required reviewers:                     │
│   @team-lead                            │
│   @senior-dev                           │
│                                         │
│  [ ✅ Approve and deploy ]  [ ❌ Reject ]│
└─────────────────────────────────────────┘
```

&nbsp;

## 승인자에게 가는 알림

&nbsp;

```
GitHub 알림 (이메일 + 웹):
  "deploy-production is waiting for your review"
  
Slack 알림 (위에서 설정):
  "🚀 스테이징 배포 완료. 확인 후 승인해주세요."
  [스테이징 확인하기] [GitHub에서 승인하기]
```

&nbsp;

## 승인 프로세스

&nbsp;

```
1. 팀장이 Slack 알림 받음
2. 스테이징 URL에서 직접 확인
3. 문제 없으면 GitHub에서 [Approve and deploy] 클릭
4. 클릭하는 순간 프로덕션 배포 job이 자동 시작
```

&nbsp;

**버튼 하나다.** 복잡한 게 아니다.

&nbsp;

&nbsp;

---

&nbsp;

# 6. Step 5 — 프로덕션 배포

&nbsp;

승인 버튼 누르면 자동으로 실행된다.

&nbsp;

```yaml
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Docker 이미지 태그 변경
        run: |
          docker pull registry.company.com/myapp:staging
          docker tag registry.company.com/myapp:staging registry.company.com/myapp:production
          docker push registry.company.com/myapp:production

      - name: 프로덕션 배포
        run: |
          ssh deploy@prod.company.com "
            docker pull registry.company.com/myapp:production
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp \
              -p 3000:3000 \
              --restart unless-stopped \
              registry.company.com/myapp:production
          "

      - name: 헬스체크
        run: |
          sleep 15
          for i in 1 2 3; do
            if curl -f https://myapp.company.com/health; then
              echo "헬스체크 통과"
              exit 0
            fi
            sleep 5
          done
          echo "헬스체크 실패!"
          exit 1

      - name: 헬스체크 실패 시 롤백
        if: failure()
        run: |
          ssh deploy@prod.company.com "
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp \
              --restart unless-stopped \
              registry.company.com/myapp:production-previous
          "
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text": "🚨 프로덕션 배포 실패 → 자동 롤백 완료"}'

      - name: 배포 완료 알림
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{
              "text": "✅ 프로덕션 배포 완료\nPR: #${{ github.event.number }}\n버전: ${{ github.sha }}"
            }'
```

&nbsp;

```
승인 클릭
  → 스테이징 이미지를 프로덕션 태그로 변경 (새로 빌드 안 함)
  → 프로덕션 서버에 배포
  → 헬스체크 (3회 시도)
  → 실패하면 자동 롤백 + Slack 알림
  → 성공하면 Slack 완료 알림
```

&nbsp;

**중요: 스테이징에서 검증된 동일한 이미지를 프로덕션에 올린다.** 다시 빌드하지 않는다. "스테이징에서 됐는데 프로덕션에서 안 돼요"를 방지.

&nbsp;

&nbsp;

---

&nbsp;

# 7. 전체 워크플로우 파일 (완성본)

&nbsp;

```yaml
# .github/workflows/pipeline.yml
name: Full Pipeline

on:
  pull_request:
    branches: [main]

jobs:
  # 1단계: 테스트
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run build
      - run: npm test

  # 2단계: AI 코드 리뷰
  ai-review:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: AI 리뷰
        uses: actions/github-script@v7
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          script: |
            // ... (위의 AI 리뷰 코드)

  # 3단계: 스테이징 배포
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [test, ai-review]
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myapp:${{ github.sha }} .
      - run: docker push registry.company.com/myapp:staging
      - run: ssh deploy@staging "docker pull && docker run ..."
      - run: curl -f https://staging.company.com/health
      - run: curl -X POST $SLACK_WEBHOOK -d '{"text":"스테이징 배포 완료"}'

  # 4단계: 프로덕션 배포 (승인 필요)
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production    # ← 승인 대기
    steps:
      - run: docker tag staging production
      - run: ssh deploy@prod "docker pull && docker run ..."
      - run: curl -f https://myapp.company.com/health
      - if: failure()
        run: echo "롤백!" && ssh deploy@prod "rollback"
      - run: curl -X POST $SLACK_WEBHOOK -d '{"text":"프로덕션 배포 완료"}'
```

&nbsp;

&nbsp;

---

&nbsp;

# 8. "우리는 GitHub Actions 안 쓰는데?"

&nbsp;

같은 개념이 다른 도구에도 있다.

&nbsp;

| 기능 | GitHub Actions | GitLab CI | Jenkins |
|:---|:---|:---|:---|
| 자동 테스트 | `on: pull_request` | `rules: - if: $CI_MERGE_REQUEST_ID` | Multibranch Pipeline |
| 승인 게이트 | `environment: production` | `when: manual` | Input Step |
| 알림 | Slack Webhook | Slack Integration | Slack Plugin |
| AI 리뷰 | Custom Action | Custom Job | Groovy Script |

&nbsp;

도구는 달라도 **흐름은 동일하다:** 테스트 → 리뷰 → 스테이징 → 승인 → 프로덕션.

&nbsp;

&nbsp;

---

&nbsp;

# 9. 도입 순서

&nbsp;

한 번에 다 하지 말고 단계적으로.

&nbsp;

```
1주차: lint + build 자동 실행 (가장 쉬움)
2주차: 스테이징 자동 배포 추가
3주차: Slack 알림 연동
4주차: AI 코드 리뷰 추가
5주차: 프로덕션 승인 게이트 추가
```

&nbsp;

**1주차만 해도 "PR 머지하면 빌드 깨지는" 사고를 막을 수 있다.**

&nbsp;

&nbsp;

---

&nbsp;

# 10. 비용

&nbsp;

| 항목 | 비용 |
|:---|:---|
| GitHub Actions | 무료 (공개 레포), 월 $4/user (팀) |
| AI 리뷰 (Claude API) | PR당 ~$0.05 (약 72원), 월 100PR = $5 (약 7,150원) |
| Docker Registry | GitHub Packages 무료 (500MB), ECR $0.10/GB |
| Slack | 무료 (Webhook) |
| **합계** | **월 $10~20 (약 14,300~28,600원)** |

&nbsp;

사람이 리뷰하는 시간 vs AI 리뷰 비용을 비교하면, PR 하나당 72원은 사실상 무료다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론

&nbsp;

```
개발자가 하는 일:
  1. 코드 작성
  2. PR 올리기
  3. AI 리뷰 피드백 반영
  4. (팀장) 승인 버튼 클릭

자동으로 되는 일:
  1. 린트 + 빌드 + 테스트
  2. AI 코드 리뷰
  3. 스테이징 배포
  4. 프로덕션 배포
  5. 헬스체크 + 롤백
  6. Slack 알림
```

&nbsp;

처음엔 "복잡해 보인다"고 느낄 수 있지만, 한 번 세팅하면 그 뒤로는 **PR 올리고 버튼 하나**다.

그리고 그 "한 번 세팅"은 위의 YAML 파일 하나를 `.github/workflows/`에 넣는 것이다.

&nbsp;

&nbsp;

---

CI/CD, GitHub Actions, AI코드리뷰, 자동배포, 스테이징, 프로덕션, Docker, 승인게이트, Slack, DevOps, 파이프라인, 자동화
