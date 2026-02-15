# 블로그를 시작합니다

안녕하세요! **hungrytech.dev** 기술 블로그에 오신 걸 환영합니다.

이 블로그는 순수 HTML/CSS/JavaScript로 만든 정적 블로그입니다.  
별도의 빌드 과정 없이 `git push` 만으로 즉시 배포됩니다.

---

## 기술 스택

- **GitHub Pages** — 무료 호스팅
- **marked.js** — Markdown 렌더링
- **highlight.js** — 코드 하이라이팅
- 빌드 도구 없음

---

## 새 포스트 작성 방법

### 1. Markdown 파일 작성

`posts/` 디렉토리에 `.md` 파일을 추가합니다.

```
posts/
  my-new-post.md
```

### 2. index.json에 메타데이터 추가

```json
{
  "slug": "my-new-post",
  "title": "포스트 제목",
  "date": "2026-02-15",
  "description": "포스트 한 줄 요약",
  "category": "java"
}
```

### 3. Push

```bash
git add .
git commit -m "post: 포스트 제목"
git push
```

---

## 코드 예시

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {

    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping
    public List<PostResponse> findAll() {
        return postService.findAll();
    }
}
```

---

앞으로 Spring, JPA, Kotlin, 아키텍처 등 백엔드 개발 관련 내용을 꾸준히 포스팅하겠습니다. 🚀
