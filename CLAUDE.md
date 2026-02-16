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

## _config.yml 수정 시 주의

- 수정 후 Jekyll 서버 재시작 필요 (일반 포스트는 자동 반영)
- 155줄 이후 설정은 수정하지 않기 (Chirpy 테마 동작에 필요)

## 기술 스택

- Ruby 3.3.6 (rbenv으로 관리)
- Jekyll 4.4.1
- Bundler 4.0.6
- Chirpy 테마 7.4 (gem 기반, chirpy-starter 사용)
- GitHub Actions로 자동 배포
