# [Doc-to-Code 4편] GitHub API와 자동화된 PR 생성 — Agentic Git Workflow 구현

&nbsp;

샌드박스에서 완벽하게 검증된 코드가 메모리에 존재한다고 해서 개발 프로세스가 끝나는 것은 아니다. 현대적인 소프트웨어 배포 파이프라인은 결국 Git 기반의 형상 관리 시스템에서 이루어진다. 

&nbsp;

지금까지는 개발자가 AI 생성 코드를 복사해서 에디터에 붙여넣고(Copy & Paste) 커밋을 수동으로 치는 반자동 수준이었다면, 이제는 에이전트 스스로가 하나의 독립적인 엔지니어(Contributor)로서 GitHub에 브랜치를 따고, 의미론적(Semantic)으로 커밋을 묶으며, 리뷰어를 지정한 PR(Pull Request)을 여는 완전한 **Agentic Git Workflow**를 구축해야 한다. 본 글에서는 GitHub의 Octokit API를 활용하여 봇(Bot)의 다중 파일 커밋 트랜잭션과 맥락이 담긴 PR 자동화 파이프라인을 심층 해부한다.

&nbsp;

&nbsp;

---

&nbsp;

# 1. Octokit을 활용한 Git Tree 제어

&nbsp;

여러 개의 파일(`controller`, `service`, `dto` 등)을 수정한 뒤 이를 로컬 디스크에 임시 저장하고 `git push`를 날리는 셸(Shell) 명령어 방식은, 동시 다발적으로 여러 에이전트가 돌고 있는 클라우드 환경에서는 컨플릭트(Conflict) 지옥을 유발한다. 
진정한 클라우드 네이티브 에이전트는 로컬 파일 시스템을 거치지 않고, GitHub의 **Git Database API**를 직접 호출하여 메모리 상에서 커밋 트리를 생성하고 푸시해야 한다.

&nbsp;

## 1-1. 메모리 기반 다중 파일 커밋 트랜잭션
단일 커밋에 3개의 파일을 동시에 올리기 위해서는 Blob 생성 → Tree 갱신 → Commit 생성 → Reference 업데이트의 4단계 API 호출이 필요하다.

&nbsp;

```javascript
import { Octokit } from "@octokit/rest";
const octokit = new Octokit({ auth: process.env.GITHUB_BOT_TOKEN });

async function createCommit(owner, repo, branch, files, commitMessage) {
  // 1. 최신 main 브랜치의 커밋 SHA 획득
  const { data: ref } = await octokit.git.getRef({ owner, repo, ref: `heads/${branch}` });
  const latestCommitSha = ref.object.sha;

  // 2. 현재 트리의 베이스 SHA 획득
  const { data: commit } = await octokit.git.getCommit({ owner, repo, commit_sha: latestCommitSha });
  const baseTreeSha = commit.tree.sha;

  // 3. 변경할 파일들을 Blob으로 만들고 새 Tree 배열 구성
  const treeEntries = await Promise.all(files.map(async (file) => {
    // 코드를 GitHub에 Blob 형태로 업로드
    const { data: blob } = await octokit.git.createBlob({ owner, repo, content: file.content, encoding: 'utf-8' });
    return { path: file.path, mode: '100644', type: 'blob', sha: blob.sha };
  }));

  // 4. 새로운 Tree 생성
  const { data: newTree } = await octokit.git.createTree({ owner, repo, base_tree: baseTreeSha, tree: treeEntries });

  // 5. 새 Commit 객체 생성 (부모는 기존 커밋)
  const { data: newCommit } = await octokit.git.createCommit({
    owner, repo, message: commitMessage, tree: newTree.sha, parents: [latestCommitSha]
  });

  // 6. 브랜치 참조(Reference) 업데이트 (Fast-forward)
  await octokit.git.updateRef({ owner, repo, ref: `heads/${branch}`, sha: newCommit.sha });
}
```

&nbsp;

이 방식은 로컬 디스크 I/O 없이 100% 네트워크 API만으로 작동하므로, 에이전트는 AWS Lambda 같은 Stateless 환경에서도 초당 수십 개의 안전한 커밋을 병렬로 날릴 수 있다.

&nbsp;

&nbsp;

---

&nbsp;

# 2. 브랜치 네이밍과 커밋 메시지 컨벤션

&nbsp;

에이전트는 멍청하게 `update_files` 같은 커밋 메시지를 남기면 안 된다. 1편에서 파싱했던 블로그 포스트의 프론트마터(Frontmatter) 데이터를 활용하여 사내 컨벤션에 맞는 엄격한 명명 규칙을 적용한다.

&nbsp;

- **Branch Name**: `feat/bot-{slug}` (예: `feat/bot-add-point-system`)
- **Commit Message**: Conventional Commits 규칙 준수.
  ```text
  feat(payment): 포인트 결제 API 및 서비스 레이어 구현
  
  - PointController: 잔액 조회 및 결제 차감 엔드포인트 추가
  - PointService: 차감 로직 데드락 방지 락 적용
  - CreatePointDto: 유효성 검사 파이프라인 추가
  
  Ref: Blog Spec #12
  ```
  에이전트는 자신이 수정한 파일의 목록을 기반으로 위와 같은 서술형 커밋 메시지를 자체 LLM 체인을 통해 동적으로 생성해 낸다.

&nbsp;

&nbsp;

---

&nbsp;

# 3. AI Generated Pull Request 본문 작성

&nbsp;

PR(Pull Request)은 인간 리뷰어가 이 코드를 심사하기 위한 핵심 컨텍스트다. 에이전트가 만든 PR 본문에는 "이 코드가 왜 만들어졌는가"에 대한 완벽한 명세서가 링크되어야 한다.

&nbsp;

```javascript
async function openPullRequest(owner, repo, branch, blogUrl, specs) {
  const prBody = `
## 🤖 AI Agent Generated PR

이 PR은 기술 블로그 명세서 기반의 자동화 파이프라인에 의해 생성되었습니다.

### 📝 원본 명세서 (Executable Spec)
- [블로그 포스트 보러가기](${blogUrl})

### ✨ 주요 변경 사항
${specs.endpoints.map(ep => `- **${ep.method}** \`${ep.path}\` 구현`).join('\n')}

### 🛡️ 샌드박스 테스트 통과 목록 (Acceptance Criteria)
${specs.acceptanceCriteria.map(ac => `- [x] ${ac.text}`).join('\n')}

---
**리뷰어 가이드**: 
이 PR은 Vercel 컨테이너 환경에서 모든 단위 테스트를 통과했습니다. 
코드의 비즈니스 로직 적합성을 위주로 리뷰해 주시고, 수정이 필요하다면 코멘트를 남겨주세요. 에이전트가 자동으로 코드를 재수정하여 커밋합니다.
  `;

  await octokit.pulls.create({
    owner, repo,
    title: `feat: ${specs.epic} 자동 생성`,
    head: branch,
    base: 'main',
    body: prBody
  });
}
```

&nbsp;

이 PR 화면을 열어본 리뷰어는 이 코드가 어떤 블로그 글을 바탕으로 기획되었는지, 그리고 사전에 정의된 테스트 시나리오를 모두 통과했는지를 한눈에 파악할 수 있다. 완벽한 추적성(Traceability)의 확보다.

&nbsp;

&nbsp;

---

&nbsp;

# 결론: 코드 생산의 파이프라인화

&nbsp;

블로그에 글을 썼을 뿐인데, 5분 뒤 내 GitHub 레포지토리에 테스트 코드가 완비된 수천 줄의 PR이 아름다운 설명과 함께 올라와 있는 마법. 이것은 단순한 자동화를 넘어 프론트엔드/백엔드 개발의 파이프라인 혁명이다.

&nbsp;

에이전트는 이제 우리의 명령을 받는 터미널 속 봇이 아니다. 우리와 함께 Git Repository를 공유하고, 컨벤션을 지키며, PR을 올리고 승인을 기다리는 완벽한 가상의 동료(Virtual Co-worker)로 진화했다.

&nbsp;

&nbsp;

---

&nbsp;

# 다음 편 예고

&nbsp;

> **[Doc-to-Code 5편] 실시간 피드백 루프 — PR 코멘트를 통한 에이전트와의 페어 프로그래밍**

&nbsp;

아무리 AI가 똑똑해도 사람의 비즈니스 의도를 100% 만족시킬 수는 없다. 에이전트가 올린 PR 코드에 시니어 개발자가 "이 부분은 레디스 캐시를 타게 수정해 줘"라고 GitHub 리뷰 코멘트를 남기는 순간! GitHub Webhook이 에이전트를 다시 깨우고, 에이전트가 코멘트 맥락을 분석하여 코드를 수정한 뒤 동일한 PR 브랜치에 추가 커밋을 밀어 넣는 '인간과 AI의 실시간 페어 프로그래밍' 아키텍처의 최종장을 장식한다.

&nbsp;

&nbsp;

---

&nbsp;

DocToCode, AI개발, Octokit, GithubAPI, PR자동화, GitWorkflow, LLMOps, 백엔드자동화
