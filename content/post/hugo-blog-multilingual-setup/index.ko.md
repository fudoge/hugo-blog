---
title: "Hugo 블로그에서 다국어 세팅하기"
description: "Hugo Blog Multilingual Guide"
date: 2025-07-01T22:01:36+09:00
image: "hugo-logo.png"
math: true
license: 
hidden: false
comments: true
draft: false
slug: "hugo-blog-multilingual-setup"
categories:
    - Blog
    - Gohugo
tags:
    - Blog
    - Gohugo
    - Cloudflare Pages
---

## 🧩 기본적인 설정 파일 구성

다국어 블로그 운영을 당장 해놓을 생각이 없더라도, 미리 구조를 잡아 놓는 것이 편할 수도 있다.

이 게시글은 ([Jimmy의 Stack 테마](https://stack.jimmycai.com/config/site))를 기준으로 하였다.

이 게시글은 한국어를 기본으로, 부가적으로 영어를 설정한다.
`config/_default/config.toml`을 다음과 같이 수정해주었다.
```toml
# Change baseurl before deploy
baseurl = "https://riveroverflow.pages.dev"
title = "Chaewoon's Blog"

# Theme i18n support
# Available values: en, fr, id, ja, ko, pt-br, zh-cn, zh-tw, es, de, nl, it, th, el, uk, ar
defaultContentLanguage = "ko"

# Set hasCJKLanguage to true if DefaultContentLanguage is in [zh-cn ja ko]
# This will make .Summary and .WordCount behave correctly for CJK languages.
hasCJKLanguage = true

[languages]
    [languages.en]
        languageCode = 'en-US'
        languageName = 'English'
        weight = 1
        [languages.en.params.sidebar]
            subtitle = 'Not just filling - overflowing'

    [languages.ko]
        languageCode = 'ko-KR'
        languageName = '한국어'
        weight = 2
        [languages.ko.params.sidebar]
            subtitle = '넘치게 채우기'


[pagination]
pagerSize = 5

```
- 현재 기본 언어를 한글로 하여 `defaultContentLanguage = "ko"`로 해주었고,
한중일 중 하나의 언어를 기본으로 한다면, `hasCJKLanguage = true`로 설정해야 한다.
- `weight`는 언어 리스트의 순서가 된다.
- `languageName`은 언어 리스트에서 보이는 언어의 이름이 된다.
- `subtitle`은 사이드바에 보일 블로그 주인의 프로필 메시지라고 보면 된다.
- 언어마다 다르게 페이지의 제목을 변경하고 싶다면, `languages.ko.title`에 이름을 지정해주면 된다.
예를 들면, 아래와 같다:



```toml
    [languages.ko]
        languageCode = 'ko-KR'
        languageName = '한국어'
        weight = 2
        title = 'Chaewoon의 블로그'
        [languages.ko.params.sidebar]
            subtitle = '넘치게 채우기'
```


---
## 📝 다국어 페이지 작성법
글을 작성할 때, 아래와 같은 구조를 사용해야 한다:

![Post Directory Structure](00_multilingual_directory.png)

```plaintext
content/
├── post/
│   └── multilingual-test/
│       ├── index.en.md
│       └── index.ko.md
```
`content/post/글제목/index.<언어코드>.md`의 형식을 지켜야 한다.

그리고, 게시글의 slug를 통해서 같은 번역글끼리 바인딩된다.  
`/content/post/multilingual-test/index.en/md`에 다음과 같이 작성해보자.
```markdown
---
title: Multilingual Test
slug: "Multilingual Test"
date: 2025-06-03
---

## This is English Article!

```

`content/post/multilingual-test/index.ko.md`에 다음과 같이 작성해보자:
```markdown
---
title: 다국어 테스트
slug: "Multilingual Test"
date: 2025-06-03
---

## 한국어 게시글입니다!

```

테스트 서버를 켜보자.
```bash
hugo server -D
```


![Korean Version of the Page](01_korean_article.png)

게시글을 들어가서, English를 눌러보면, 이렇게 번역 페이지로 이동한 것을 볼 수 있다!   
![English Version of the Page](02_english_article.png)

---
## 🎭 게시글 이외의 영역들에도 다국어 세팅해두기
영어 번역 페이지에는 사이드바다 비어있는 것을 볼 수 있다.
page, content, 등등의 home역할을 하는 마크다운 파일들도 다국어 세팅을 각각 해줘야만 한다.  
categories또한 마찬가지이다.

Pages가 무엇인지와, categories가 무엇인지에 대해서는 다음 게시글에서 확인할 수 있다.  
일단은 현재 있는 것들에 대해서만 번역 레플리카를 만들어주자.

아래와 같이 Content내부의 Markdown 파일들도 다국어로 구성해주면 된다!  
![Content_directory](03_content_directory.png)

만약 더 추가적인 설정이 하고 싶다면, `i18n/<언어코드>.toml`을 만들어서 기존 설정을 오버라이드 하면 된다.  
기존 i18n 파일들의 설정은 [여기](https://github.com/CaiJimmy/hugo-theme-stack/tree/master/i18n)에서 확인가능하다.  

---
## 📚 References
[Stack by Jimmicai - config/i18n](https://stack.jimmycai.com/config/i18n)  
[Hugo Documnet - content management/multilingual](https://gohugo.io/content-management/multilingual/)  
[Hugo Document - configuration/languages](https://gohugo.io/configuration/languages/)
