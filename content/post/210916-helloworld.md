---
title: "Hello World"
date: 2021-09-15T21:52:55+09:00
description: "Example article description"
categories:
  - "blog"
tags:
  - "hugo"
  - "blog"
  - "github pages"
menu: main # Optional, add page to a menu. Options: main, side, footer

# Theme-Defined params
# thumbnail:  "img/placeholder.png" # Thumbnail image
lead: "다시 블로그를 시작하며" # Lead text
comments: false # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
pager: true # Enable pager navigation (prev/next) for specific page
toc: true # Enable Table of Contents for specific page
mathjax: false # Enable MathJax for specific page
sidebar: "right" # Enable sidebar (on the right side) per page
widgets: # Enable sidebar widgets in given order per page
  - "search"
  - "recent"
  - "categories"
  - "taglist"
---

## Hello World

블로그에 발자취를 남기지 않은 지도 벌써 3년이나 됐다.

3년간 블로그는 쉬었지만 나름대로 열심히 일하면서 많은 걸 배우고 느낀 것도 많았지만 블로그를 잠시 쉬다 보니 어느덧 3년이라는 시간이 지나버렸고 게다가 3년이라는 시간을 기술적으로 어떻게 보냈는지 잘 정리가 되지 않는다.

넘치는 새로운 오픈소스의 시대에서 살아남은 좋은 솔루션들이 이제는 자리 잡고 있어 뭘 잡고 공부해도 좋은 시절인데도 불구하고 그런 기술들을 제대로 쫓아가지도 못하고 있는 현실을 크게 깨달으면서 이제는 쉬어갈 수 없다는 생각이 들었다.

## 어디에서 시작해야 할까?

기존에는 티스토리를 사용했지만, 이제는 조금 더 불편한 시스템에서 내 손으로 더 고생하면서 글을 쓰고 싶다. 

프론트엔드 개발자도 아니고 테마를 가져다 쓰면서 수정하기도 벅차겠지만 당시 티스토리를 선택했던 이유도 당시에는 커스터마이징이 충분히 많이 가능하다고 생각했기 때문이다. 

그래서 이제는 가장 불편한 방식으로 가장 편하게 글을 쓰면서 뭔가를 더 배우고 싶다.

그러다 보니 Gihtub Pages를 선택하게 됐다.

## 이제는 이렇게 쓰고 싶다.

예전에는 글을 써서 누군가에게 뭔가를 많이 알려주고 싶었다.

그러다 보니 글을 하나 쓰는데도 꽤 오랜 시간이 걸리기도 했다. 하지만 이제는 그런 의미보다 조금은 발자취처럼 기술적인 내용을 쓰고 싶다. 하지만 이런 생각이 있더라도 원래 글을 쓰던 방식으로 돌아갈 가능성이 높을 것 같다.

이런 바람이 있다보니 이제는 일단 Notion으로 이것저것 빠르게 정리를 하고 Notion에서 Export 기능을 사용해서 Markdown을 추출한 뒤 Github pages로 글을 올리면 가장 편할 것 같다는 생각이 들었다.

## 왜 Hugo인가?

github pages를 꾸밀 때 [jekyll](https://jekyllrb.com/), [hexo](https://hexo.io/ko/index.html), [hugo](https://gohugo.io/), [ghost](https://ghost.org/) 등 많은 방법이 있는데

- 윈도우와 맥에서 사용이 비슷하거나 편할 것
- 블로그를 위한 레퍼지토리와 브랜치 관리가 편할 것

위 두 가지를 고민해 봤을 때, 일단 jekyll은 윈도우에서 사용이 어렵다. hexo는 이미 한번 경험해본 것이라 제외. 

Ghost와 Hugo가 가장 고민이 됐었는데 Hugo가 Go 라는 점에서 끌리기도 했고, [Hugo의 Github Pages 매뉴얼](https://gohugo.io/hosting-and-deployment/hosting-on-github/)을 봐도 로컬에서 블로그를 다시 올리고 퍼블리싱하는데 있어서 가장 빠르고 편해보였다.

휴고의 경우 main 브랜치와 publish 용 브랜치만으로 충분히 블로그 관리가 가능해 보였다.
[Ghost의 경우 Buster를 같이 설치해야 하기 때문](https://stefanscherer.github.io/setup-ghost-for-github-pages/)에 덜 매력적으로 보인것도 있었다.

게다가 [main브랜치에 푸시하면 자동으로 내용을 반영해주는 github action도 ](https://github.com/peaceiris/actions-hugo) 있어 내가 추구하는 가장 이상적인 방법에 가까웠다.

이렇게 할 경우 아래와 같은 절차로 글을 쓸 수 있게 된다.
- 글을 작성한다.
- `hugo server -D` 로 서버를 실행한다.
- `http://locahost:1313` 주소로 접속해 글을 확인하고 실시간으로 수정한다.
- main 브랜치로 푸시한다.
- github action을 통해 자동으로 github pages에 반영된다.

안정될때까지 테마를 계속 건드릴것 같아 일단은 매우 단순한 테마를 적용해서 시작해 보기로 했다.