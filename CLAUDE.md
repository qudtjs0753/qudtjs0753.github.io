# 블로그 운영 가이드

## 글 작성/배포 워크플로우

```
1. _posts/ 폴더에 새 글 작성 (YYYY-MM-DD-제목.md)
2. 로컬에서 확인: bundle exec jekyll serve
3. git add → git commit → git push
4. GitHub Actions가 자동으로 빌드 & 배포 (약 1~2분)
5. https://qudtjs0753.github.io 에서 확인
```

## 자주 사용할 명령어

```bash
# 로컬 미리보기
bundle exec jekyll serve

# 변경사항 배포
git add .
git commit -m "커밋 메시지"
git push

# 빌드 상태 확인
gh run list --repo qudtjs0753/qudtjs0753.github.io --limit 3

# 빌드 로그 확인 (실패 시)
gh run view <run-id> --repo qudtjs0753/qudtjs0753.github.io --log
```

## 포스트 파일명 규칙

- 위치: `_posts/` 폴더
- 형식: `YYYY-MM-DD-제목.md`
- 영어 소문자 + 하이픈 권장

## 포스트 front matter 규칙

- **categories**: 반드시 하나만 지정한다 (예: `categories: [Vim]`)
- 기존 카테고리 중 맞는 것이 있으면 사용하고, 없으면 새로 만든다
- **tags**: 카테고리와 겹치지 않게 지정한다 (예: 카테고리가 `Vim`이면 태그에 `vim`을 넣지 않는다)


## 블로그 글 작성 시 개인정보 주의

- 블로그 포스트에 실제 개인 폴더 경로(예: `/home/helloworld/`, `/home/사용자명/workspace/` 등)를 절대 포함하지 않는다
- 경로 예시가 필요하면 `~/dotfiles`, `/home/user/dotfiles` 같은 일반적인 표현을 사용한다


## 글 구조 규칙

- 글 상단에 **변경 후의 최종 결과**(스크린샷, 요약 테이블 등)를 먼저 보여준다
- 각 섹션은 **설치 → 설정 → 부연설명** 순서로 작성한다
  - 설치/설정 코드를 먼저 제시하고, 원리나 배경 설명은 그 아래에 배치한다


## _config.yml 수정 시 주의

- 155줄 이후 설정은 수정하지 않기 (Chirpy 테마 동작에 필요)

## 기술 스택

- Ruby 3.3.6 (rbenv으로 관리)
- Jekyll 4.4.1
- Bundler 4.0.6
- Chirpy 테마 7.4 (gem 기반, chirpy-starter 사용)
- GitHub Actions로 자동 배포
