# 기술 블로그: alswns1201.github.io

Jekyll(`jekyll-theme-chirpy` ~> 7.6) 기반 GitHub Pages 블로그. 포트폴리오/면접용이라
"면접관이 봤을 때 고민을 많이 한 글"로 읽혀야 한다. (티스토리 이관은 2026-10 완료)

사이트 소스는 전부 `blog/` 안에 있다. 배포 워크플로(`.github/workflows/pages.yml`)가 `blog/`에서 빌드한다.

## 글 쓰기 규칙
- 파일: `blog/_posts/YYYY-MM-DD-영문-slug.md`, 주소는 `/posts/영문-slug/`.
- front matter:
  ```yaml
  ---
  title: "제목"
  date: 2026-10-24 10:00:00 +0900
  categories: [인프라]
  ---
  ```
  `layout: post`는 `blog/_config.yml` defaults로 자동 적용되니 안 써도 된다.
- 카테고리는 하나만 쓴다. 지금 쓰는 카테고리는 `grep -rh "^categories:" blog/_posts | sort | uniq -c`로 확인한다.
  새 카테고리가 필요하면 사용자에게 먼저 묻고, 추가하면 `blog/_includes/topics.html`의 priority에도 넣는다.
- 이미지: `blog/assets/img/posts/`.
- 글을 합치거나 주소를 바꾸면 새 글에 `redirect_from:`으로 예전 주소를 연결한다 (`jekyll-redirect-from`).

## 내용 원칙
- 없던 실무 경험담은 지어내지 않는다. 대신 왜 그런지, 트레이드오프, 흔한 실수 같은 기술적인 깊이를 더한다.
- 회사 업무를 기반으로 쓴 글에는 회사를 알아볼 수 있는 정보를 넣지 않는다. 회사명·서비스명,
  리소스 이름(Lambda/큐/테이블), API 경로, 업무 고유 기능, 계정·시크릿이 여기에 해당한다.
  이름은 일반화해서 "예시 이름" 안내를 달고, 사내 다이어그램은 원본과 다른 레이아웃으로 다시 그린다.
- 회사 업무 기반 글을 올린 뒤에는 경력기술서 누적본도 갱신한다 (`career-log` 스킬).

## 빌드/배포
- `gh`/`git`으로 커밋·push (계정 `alswns1201`). push 전에는 사용자에게 확인한다.
- 로컬 Ruby가 2.6이라 `jekyll build`가 안 된다. 빌드 결과는 push한 뒤 GitHub Actions(`.github/workflows/pages.yml`)로 확인한다.
- 테마 색/폰트는 `blog/assets/css/jekyll-theme-chirpy.scss`에서 바꾼다. 테마는 `theme_mode: light` 고정이고,
  아이콘 폰트는 `'Font Awesome 7 Free'`를 쓴다.

## 비공개 폴더
- `경력기술서_누적/` (회사 실명 포함. `.gitignore`로 제외되어 있으니 절대 커밋하지 않는다. 관리는 `career-log` 스킬로 한다)
  - `경력기술서_엑심베이_누적.md`: 경력기술서에 쓸 만한 내용 (→ PDF)
  - `업무기록_엑심베이.md`: 기록용 메모 (날짜순)
  - 사용자가 업무 내용을 주면 둘 중 하나로 나눠 넣는다. 기존 경력기술서와 합치는 건 요청할 때만 한다.
