---
layout: post
title: "GitHub 블로그 구축 트러블 슈팅 - gh-pages"
date: 2024-12-17 00:00:00 +0900
categories: [GitHub, Jekyll]
tags: [jekyll, github pages, troubleshooting]
description: "Jekyll에서 GitHub Pages를 사용할 때 발생한 문제를 해결한 과정 기록"
---

# GitHub 블로그 구축 트러블 슈팅 - gh-pages

### 문제 시작: index.html에서 home.html을 불러오기 실패

Jekyll에서 Chirpy Theme을 사용해서 블로그를 만들어 보려고 했다. GPT와 함께 만들어 가던 중, 배포 페이지에
```
--- layout: home title: Welcome ---
```
이 글씨만 나오는 문제가 있었다. 문제를 해결하는 과정을 기록하기 위해 글을 작성한다.

먼저 master 브랜치로만 구성하면 될 줄 알았지만 그렇지 않았다. gh-pages 브랜치를 추가로 만들어서 해당 브랜치를 source 브랜치로 사용해야 했다. 왜 그럴까?

1.	Jekyll의 빌드 과정 이해

Jekyll은 소스 파일(_layouts, _posts, _config.yml 등)을 기반으로 _site라는 디렉토리에 정적 파일(HTML, CSS, JS 등)을 생성한다.

2.	GitHub Pages 동작 원리

	•	GitHub Pages는 기본적으로 Jekyll 빌드 기능을 내장하고 있다.

	•	브랜치를 master 또는 gh-pages로 설정하면, 해당 브랜치의 내용을 기준으로 배포를 진행한다.

	•	master 브랜치: 소스 파일을 올리면 GitHub이 자동으로 빌드해서 정적 파일을 생성.

	•	gh-pages 브랜치: 빌드된 결과물인 정적 파일만 존재해야 하며, GitHub은 이를 그대로 서빙한다.

3.	왜 gh-pages 브랜치가 필요할까?

master 브랜치에 Jekyll 소스 파일이 있는 경우 GitHub이 자동으로 빌드를 수행하지만, 이 과정에서 로컬과 GitHub 빌드 환경의 차이로 오류가 발생할 수 있다.

그래서 로컬에서 빌드한 결과물만 gh-pages 브랜치에 업로드하면, GitHub이 별도의 빌드를 수행하지 않고 _site의 정적 파일을 그대로 서빙하게 된다.


아래는 최종 정리한 내용이다.

## 1. 문제 상황

	•	블로그 화면에 --- layout: home title: Welcome ---라는 원시 텍스트가 렌더링됨.

	•	의도한 홈페이지 디자인이 표시되지 않음.

## 2. 원인 분석

	•	GitHub Pages가 Jekyll 사이트를 제대로 빌드하지 못했음.

	•	gh-pages 브랜치에 빌드된 결과물(정적 파일)이 있어야 하지만, 소스 파일과 혼동되었음.

	•	_layouts, _includes 등 Jekyll의 템플릿 파일이 포함되지 않은 상태에서 index.html이 제대로 렌더링되지 않음.

## 3. 해결 과정

### Step 1: 로컬 개발 환경 설정

#### 1.	Jekyll 및 필요한 의존성 설치:
```
bundle install
```

#### 2.	로컬 빌드 확인:
```
bundle exec jekyll serve
```
	•	127.0.0.1:4000에서 블로그가 정상적으로 동작하는지 확인.

	•	로컬에서는 문제가 없었음.

### Step 2: GitHub Pages 브랜치 설정

#### 1.	GitHub Pages 설정 확인:
	•	Repository > Settings > Pages에서 Source Branch를 확인.
	•	master 브랜치를 설정하면 GitHub이 자동으로 Jekyll 빌드를 수행.
	•	gh-pages 브랜치를 사용할 경우, 빌드된 정적 파일만 업로드해야 함.
#### 2.	수동 배포로 전환:
	•	master 브랜치에서 Jekyll로 빌드된 _site 디렉토리를 gh-pages 브랜치로 옮김.

```
bundle exec jekyll build
git checkout gh-pages
git rm -rf .
cp -r _site/* .
git add .
git commit -m "chore: deploy site to gh-pages"
git push origin gh-pages
```

### Step 3: GitHub Actions를 통한 자동 배포 설정

#### 1. .github/workflows/deploy.yml 파일 생성:

	•	GitHub Actions를 사용해 master 브랜치에 커밋될 때 자동으로 빌드하고 gh-pages에 배포.
```
name: Deploy Jekyll Site

on:
  push:
    branches:
      - master

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.1

      - name: Install Dependencies
        run: |
          gem install bundler
          bundle install

      - name: Build Site
        run: |
          bundle exec jekyll build

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./_site
```

#### 2.	Workflow 실행:

	•	master 브랜치에 커밋을 푸시하면 GitHub Actions가 자동으로 실행됨.

	•	gh-pages 브랜치에 빌드된 결과물이 배포됨.

#### 3.	GitHub Pages 설정:

	•	Settings > Pages에서 Source를 gh-pages 브랜치로 변경.

### Step 4: 빌드 오류 수정
#### 1.	HTML-Proofer를 사용해 오류 확인:
```
bundle exec htmlproofer ./_site --allow-hash-href --disable-external
```

	•	존재하지 않는 이미지와 스크립트 경로 오류 확인.
	•	누락된 파일들을 로컬에 추가하거나 참조 경로 수정.

#### 2.	Favicon과 프로필 사진 추가:

	•	assets/img 디렉토리에 favicon.ico, apple-touch-icon.png 등 필요한 이미지 업로드.

	•	_config.yml에 프로필 이미지 경로 수정:

```
avatar: "/assets/img/avatar.jpg"
```

#### 3.	정적 파일 재배포:
```
bundle exec jekyll build
git checkout gh-pages
git rm -rf .
cp -r ../_site/* .
git add .
git commit -m "fix: update missing assets and scripts"
git push origin gh-pages
```
### Step 5: 최종 검증
#### 1.	로컬 테스트:
```
bundle exec jekyll serve
```

#### 2.	GitHub Actions 실행 확인:
	•	Actions 탭에서 Workflow가 성공적으로 완료되었는지 확인.

	3.	웹사이트 확인:

	•	https://username.github.io에 접속하여 정상적으로 렌더링되는지 확인.

### 4. 결론

	•	--- layout: home title: Welcome --- 이 표시된 이유는 gh-pages 브랜치에 빌드된 결과물이 아닌 소스 파일이 업로드되었기 때문.

	•	GitHub Actions를 사용해 자동으로 빌드 및 배포하도록 설정해 문제를 해결함.

	•	이제 블로그를 수정할 때 master 브랜치에 변경 사항을 푸시하면 GitHub Actions가 자동으로 빌드하여 gh-pages에 배포됨.