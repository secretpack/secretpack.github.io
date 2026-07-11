# Secretpack 블로그 고도화 설계

- 날짜: 2026-07-11
- 대상 저장소: `secretpack/secretpack.github.io` (GitHub Pages 사용자 사이트)
- 테마: Minimal Mistakes 4.24.0
- 상태: 승인됨

## 배경 / 현재 상태

Minimal Mistakes 테마를 통째로 클론한 "순정 테마" 상태. 실제 블로그로서의 기능이 없음:

- `_posts` 없음 → 글 0개
- `_pages` 없음 → About/아카이브 등 페이지 없음
- `index.html`은 `layout: home`만 있고 보여줄 글이 없어 빈 홈
- `_data/navigation.yml`이 테마 제작자 데모 사이트(mmistakes.github.io) 링크만 가리킴
- `docs/`(18MB 테마 데모), `test/`, `CHANGELOG.md`, `screenshot*.png` 등 테마 개발 잔여물 다수
- 검색/댓글/애널리틱스 전부 비활성

`_config.yml`의 사이트 메타(제목 "Secretpack's Blog", locale ko-KR, author 프로필, paginate 5)는 이미 커스터마이즈됨.

## 목표

바로 사용할 수 있는 보안/개발 리서치 블로그 상태로 만들기. 로컬 미리보기까지 확인.

## 설계

### 1. 저장소 정리 + remote_theme 전환

테마 소스를 저장소에서 제거하고 원격 테마로 전환한다.

- **삭제**: `_layouts/`, `_sass/`, `_includes/`(테마 기본), `docs/`, `test/`, `CHANGELOG.md`,
  `screenshot.png`, `screenshot-layouts.png`, `banner.js`, `Rakefile`, `staticman.yml`,
  `.travis.yml`, `minimal-mistakes-jekyll.gemspec`, `package.json`, `package-lock.json`
- **보존**: 본인 이미지(`avatar.jpg`, `avatar_remake.jpg`, `logo.png`, `secretpack.png` 등)를
  `assets/images/`로 이동하고 `_config.yml` 경로를 그에 맞게 정리
- **Gemfile**: gemspec 의존 제거, `github-pages` gem + `jekyll-include-cache` 사용
- **_config.yml**: `remote_theme: "mmistakes/minimal-mistakes@4.24.0"` 추가
- 결과: 저장소 ~18MB → ~1MB 이하

### 2. 필수 페이지 (`_pages/`)

- `about.md` — 보안 리서처 소개 (author 정보 기반)
- `category-archive.md` — `layout: categories`, permalink `/categories/`
- `tag-archive.md` — `layout: tags`, permalink `/tags/`
- `year-archive.md` — `layout: posts`, permalink `/year-archive/`
- `404.md` — 404 페이지
- 검색은 페이지가 아니라 `_config.yml`의 `search: true`로 마스트헤드 아이콘 생성

### 3. 네비게이션 (`_data/navigation.yml`)

데모 링크 제거 후 본인 메뉴로:
`Home(/) · Categories(/categories/) · Tags(/tags/) · Year Archive(/year-archive/) · About(/about/)`

### 4. 첫 글 + 홈

- `_posts/2026-07-11-welcome-to-secretpack-blog.md` — 마크다운/코드블록/카테고리·태그 예시가 담긴
  샘플 글 1개 (홈·아카이브·태그·검색 동작 검증용)
- `index.html`은 `layout: home` 유지 → 최근 글 목록 표시 확인

### 5. 기능 활성화 (`_config.yml`)

- **검색**: `search: true`, `search_provider: lunr` (무료, 클라이언트 사이드)
- **댓글**: `comments.provider: giscus`. `repo_id`/`category_id`는 플레이스홀더로 두고,
  사용자가 GitHub에서 (1) 저장소 Discussions 활성화 (2) giscus 앱 설치 (3) 값 획득 후 제공하면 연결
- **스킨**: `minimal_mistakes_skin: "dark"`
- **SEO/피드/사이트맵**: `jekyll-seo-tag`, `jekyll-feed`, `jekyll-sitemap` 활성 (GitHub Pages 기본)
- **아카이브**: `category_archive`/`tag_archive` = `liquid` (기존 설정 유지) → 페이지로 연결
- **포스트 defaults**: author_profile, read_time, comments, share, related, TOC 켜기

### 6. 로컬 미리보기

- `winget install RubyInstallerTeam.RubyWithDevKit`
- `gem install bundler` → `bundle install` → `bundle exec jekyll serve`
- `http://localhost:4000` 에서 렌더링 확인

### 배포

GitHub Pages 사용자 사이트라 `main` push 시 자동 빌드. **push는 마지막에 사용자 승인 후** 진행.

## 리스크 / 유의점

- **커밋 계정**: git user 정보 비어 있음 → 커밋 전 이름/이메일 설정 필요
  (기본값: `secretpack` / 세션 이메일 `secretmadb33pack@gmail.com`, 사용자 확정 대기)
- **giscus**: Discussions 활성화·앱 설치는 사용자 GitHub UI 액션 필요. 나머지는 전부 자동 세팅
- **한글 검색**: lunr는 한국어 형태소 분석은 없으나 단어 단위 검색은 정상 동작

## 완료 기준

- `bundle exec jekyll serve`가 에러 없이 빌드되고 localhost:4000에서 홈/글/아카이브/검색이 뜬다
- 저장소에 테마 잔여물이 없다
- 네비게이션이 본인 페이지를 가리킨다
- push는 사용자 승인 후 별도 진행
