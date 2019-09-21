---
title: git拉取特定的目录
date: 2019-09-21 12:12:28
tags:
- git
---

# 场景

有时候我们只需要一个git仓库的特定的目录就行了，但是去要拉取整个仓库。本文使用sparsecheckout来解决这个问题

<!--more-->

# 使用


```bash
#!/bin/bash

DIR="/apps/errors"
REPOS="git@github.com:username/repository.git"
BRANCH="gh-pages"
CHECKOUT_DIR="errors/"

mkdir -p $DIR
if [ -d "$DIR" ]; then
  cd $DIR
  git init
  git remote add -f origin $REPOS
  git fetch --all
  git config core.sparseCheckout true
  if [ -f .git/info/sparse-checkout]; then
    rm .git/info/sparse-checkout
  fi
  echo $CHECKOUT_DIR >> .git/info/sparse-checkout
  git checkout $BRANCH
  git merge --ff-only origin/$BRANCH
fi
```