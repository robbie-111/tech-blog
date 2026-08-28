# tech-blog

[Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)로 만든 기술 블로그.
`main`에 푸시하면 GitHub Actions가 빌드해 GitHub Pages로 배포한다.

**https://robbie-111.github.io/tech-blog/**

## 로컬에서 실행

```sh
git clone --recurse-submodules https://github.com/robbie-111/tech-blog.git
cd tech-blog
hugo server -D          # http://localhost:1313/
```

이미 클론한 저장소라면 테마를 먼저 받는다: `git submodule update --init --recursive`

## 글 쓰기

```sh
hugo new content posts/my-post.md
```

`content/posts/`에 마크다운을 추가하고 front matter의 `draft`를 `false`로 바꾼 뒤 푸시하면 배포된다.
