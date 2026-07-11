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

## 주제(카테고리) 컨벤션

좌측 사이드바의 **Topics 네비게이션**은 `_data/topics.yml` 의 기본 주제를 항상
모두 표시하고(글 개수 배지 포함), 글이 없는 주제는 흐리게 처리해 클릭되지 않는다.
글이 생기면 링크로 활성화되고 개수가 올라간다. 순서와 표시 이름은
`_data/topics.yml` 에서 관리하며, 기본 주제는 다음과 같다:

`Reverse Engineering` · `Pwn` · `Fuzzing` · `Web` · `Firmware` · `Mobile` · `CTF` · `Notes`

- 글을 쓸 때 `categories:` 에 위 이름을 그대로 쓰면 네비에 일관되게 쌓인다.
- `_data/topics.yml` 에 없는 새 카테고리로 글을 올리면 네비 하단에 자동으로 추가된다.
- 순서를 바꾸거나 라벨을 다르게 표시하려면 `_data/topics.yml` 만 수정하면 된다.

## 로컬 미리보기

Ruby가 설치된 상태에서:

```bash
bundle install
bundle exec jekyll serve
# http://localhost:4000
```

## 배포

`main` 브랜치에 push하면 GitHub Pages가 자동으로 빌드/배포한다.
