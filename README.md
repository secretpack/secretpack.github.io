# secretpack.github.io

[secretpack.github.io](https://secretpack.github.io) — Security Research and Development blog.

Built with [Jekyll](https://jekyllrb.com/) and the
[Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) theme (via `remote_theme`)
and hosted on GitHub Pages.

## 구조

```
_config.yml         # 사이트 설정 (테마, 검색, 댓글, author 등)
_data/
  navigation.yml    # 상단 네비게이션 메뉴
_pages/             # About, 아카이브, 404 등 고정 페이지
_posts/             # 블로그 글 (YYYY-MM-DD-title.md)
assets/             # 이미지, favicon, 커스텀 sass 진입점
_includes/head/     # <head> 커스텀 스니펫 (favicon 등)
index.html          # 홈 (layout: home)
```

## 글 작성

`_posts/` 에 `YYYY-MM-DD-제목.md` 형식으로 파일을 만든다:

```markdown
---
title: "글 제목"
categories:
  - Research
tags:
  - reversing
  - windows
---

본문...
```

## 로컬 미리보기

Ruby가 설치된 상태에서:

```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000
```

## 배포

`main` 브랜치에 push하면 GitHub Pages가 자동으로 빌드/배포한다.
