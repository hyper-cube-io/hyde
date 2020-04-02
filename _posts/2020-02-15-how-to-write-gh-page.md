---
layout: post
title: github-jekyll에 어떻게 글을 쓰나요? (feat. Markdown)
author: sh
tags: [study, gh-pages, jekyll, github]
---
> 우리가 만드는 이 블로그는 Github와 jekyll으로 구성되어 있다.<br/>
> Markdown 문서를 만들어 Github에 반영하면 글이 보인다. 이것을 jekyll을 통해 서비스를 하고 있다.<br/>
> 글을 쓰는 방법이 어렵고 쉽게 잊을 수 있어 정리하여 기록한다.<br/>

## hyper-cube.io에 글을 쓰자
우리의 blog에 글을 쓰려면 서비스 하고 있는 hyper-cube-io계정의 'hyper-cube.io' Repo에 Markdown 문서를 올리면 된다.
단, 우리는 jekyll로 서비스를 하고 있기때문에 jekyll에서 정의한 gh-pages라는 브랜치에 해당 작업이 진행되어야 한다.
우리들은 blog 관리를 위해 'hyper-cube.io' Repo를 각자 Github 계정으로 Fork하여 글을 올린 후 Pull Request를 하도록 정하였다.<br/>
`물론 'hyper-cube.io'를 로컬로 clone 후 gh-pages의 브랜치에 바로 push하여 글을 올리는 것도 가능하다.`

### hyper-cube-io에서 내 Github 계정으로 Repository Fork하기
글을 쓰기 위해서는 자신의 Github로 'hper-cube.io' Repo를 Fork 할 필요가 있다.
![fork to repo]({{site.url}}/public/posts_images/fork_to_repo.png)

### 내 Repo 로컬로 clone하여 문서 작성
일반적인 Git을 사용하듯이 Fork한 Repo를 로컬로 clone한다.
```
git clone <repo_url>
```
~~clone 후 실제 문서는 gh-pages 브랜치의 \_posts에 넣어줘야 보인다. 우리도 gh-pages 브랜치로 이동하여 작성하도록 한다.~~<br/>
clone 후 실제 작성한 문서는 master 브랜치에서 작업하며 마스터 브랜치의 \_posts에 넣어 줘서 작업한다. 이것은 'hyper-cube.io' Repo의 gh-pages에 올라와 있는 다른 사람의 문서를 건드릴 수 없게 하기 위함이다.<br/>
폴더 구조는 jekyll문서에 잘 나와 있으니 참고 하면 된다. [Directory Structure](https://jekyllrb.com/docs/structure/) 
```
git checkout master
```

현재 작성 중인 '\_posts/<문서명>.md' 문서 자체는 Markdown 형식으로 작성하며 Google에 검색하면 많은 예제가 있으니 참고 하시길..<br/>

![document write]({{site.url}}/public/posts_images/document_write.png)

이제 작성한 문서를 내 Repo에 push한다.
```
git add _posts/2020-02-15-how-to-write-gh-page.md
git add public/posts_images/fork_to_repo.png
git add public/posts_images/document_write.png

git commit -m \"문서 작성법 업로드\"
git push 
```

### hyper-cube-io에 Pull Request하기
앞에서 말했듯이 실제 페이지의 서비스는 hyper-cube-io/hyper-cube.io의 gh-pages 브랜치에서 하고 있다.
따라서 내 Repo에 올린 내용을 hyper-cube-io/hyper-cube.io에 반영 해 줘야 한다.

내 Repo에 내용들을 실제 hyper-cube-io에 반영하기 위해서는 Pull Request라는 과정을 거쳐야 한다.
![pull request]({{site.url}}/public/posts_images/pull_request.png)
![create pull request]({{site.url}}/public/posts_images/create_pull_request.png)
위 사진에서 보듯이 pull request를 생성할 때 base repository의 branch를 gh-pages로 head repository의 branch를 master로 설정 한다.
아래에는 내 repository에서 원본 repository로 반영할 commit 목록이 보인다.
이 과정은 내가 작업한 내용들이 실제 서비스에 반영해도 되는지 hyper-cube-io를 운영하는 권한 있는 사람들에게 승인을 받는다.

승인 과정이 끝나면 몇 초후에 페이지에서 제대로 반영되는지 확인한다.
