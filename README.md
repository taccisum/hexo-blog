# Blog

This is my personal blog Hexo-based.

# How to 

## Run

```shell
$ npm install
$ npm install -g hexo-cli
$ hexo s \
-g \ # re-generate
--draft # show draft
```

## Deploy

```shell
$ hexo d -g
```


如果出现 The "mode" argument must be integer. 报错，可能是 node 版本过高导致，切换下再执行就好

```shell
$ n 13.14.0
```
