---
title: "GitHub Pages×HugoでMarkdownブログを公開する方法メモ（n番煎じ）"
# publishdate: 2023-12-01
date: 2023-11-30
tags: [GitHub Pages, Hugo, Markdown]
toc: true
---


## 初期設定

<a href="https://gohugo.io/" target="_blank">Hugo</a>
と
<a href="https://git-scm.com/" target="_blank">Git</a>
を予めインストールしておく．（環境によってはGoとか必要になるらしい）

GitHubにアクセスし，予めリポジトリを新規作成．


```sh
hugo new site 新しいサイトの名前
cd 新しいサイトの名前
git init
```

<a href="https://themes.gohugo.io/" target="_blank">hugo Themes</a>
から好きなテーマを選ぶ．テーマ個別のページに飛ぶと`git submodule add ~`から始まるインストールコマンドがあるので，コピって実行する．
その後，設定ファイルにテーマ名を追記．

```sh
git submodule add https://github.com/zwbetz-gh/cupper-hugo-theme.git themes/cupper-hugo-theme
echo "theme = 'cupper-hugo-theme'" >> hugo.toml
```

`themes/テーマ名/`の中にサンプルのファイル構成が用意されている場合がある．その場合，設定ファイルごとカレントディレクトリ直下に中身をコピーする．

カレントディレクトリをGitHubにプッシュする．

GitHubのリポジトリのSetting→PagesにあるBuild and deploymentのSourceをGitHub Actionsに変更．Saveを押下．

リポジトリにある`.github/workflows/hugo.yaml`を以下の内容に編集する．

```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.120.2
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v3
      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

すべての変更が終了した後，コミットを行う．
コミット終了後数分後にWebページがアクティブになる．
GitHubのSetting>PagesあるいはActionsからURLを確認できる．

## 編集作業

以下のコマンドでサーバーが起動する．`http://localhost:1313/`あるいは`http://localhost:1313/サイト名`でアクセスできる

```sh
hugo server
```

ファイルの編集はリアルタイムでブラウザに反映される．サンプルページを色々いじって記事を書いていこう．

<br>

**参考文献**

<a href="https://gohugo.io/getting-started/quick-start/" target="_blank">Quick start | Hugo</a><br>
<a href="https://gohugo.io/hosting-and-deployment/hosting-on-github/" target="_blank">Host on GitHub Pages | Hugo</a>
