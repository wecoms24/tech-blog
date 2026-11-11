# WECOMS24 Tech Blog

https://wecoms24.github.io/tech-blog/ — GitHub Pages(Jekyll, just-the-docs 테마)로 만든 제품 기술 문서 사이트.

## 글 올리는 법

1. `docs/products/<제품>/` 아래에 마크다운 파일을 둔다. 제품 폴더가 없으면 `index.md`(아래 머리말 참고)를 먼저 만든다.
2. 파일 맨 위 머리말:

```yaml
---
title: 문서 제목
parent: <제품 폴더의 title>
grand_parent: 제품
nav_order: 2
---
```

3. 본문에서 다른 글로 링크할 때는 `{{ "/products/<제품>/<글>/" | relative_url }}`를 쓴다(`{% link %}`는 `baseurl`을 빠뜨려 404가 난다).
4. `main`에 푸시하면 GitHub Pages가 1~2분 안에 다시 만든다.

제품 폴더 `index.md` 머리말은 `title: <제품 이름>`, `parent: 제품`, `has_children: true`, `permalink: /products/<제품>/`.

## 로컬 미리보기

```bash
bundle install
bundle exec jekyll serve --baseurl /tech-blog
```
