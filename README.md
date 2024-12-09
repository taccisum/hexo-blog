# Blog source code

This is my personal blog Hexo-based.

Hexo versions:
- node: 13.14.0
- hexo: 3.7.1
- hexo-cli: 3.1.0

> Tip: Node version 13.14.0 is too much lower than latest in current time(20241209). We can use `n` to install & switch it quickly without bad effect.

# How to 

## Run

```shell
$ npm install --registry=https://registry.npmmirror.com
$ npm install -g hexo-cli@3.1.0 --registry=https://registry.npmmirror.com
$ hexo s
```

tips:
- add `--draft` option to show draft posts
- add `--config` option to specify custom config files

sample:
```shell
$ hexo s --config _config.yml,configs/next/config.yml --draft
```

## Generate static files

```shell
$ hexo g --config _config.yml,configs/next/config.yml
```

## Deploy

```shell
$ hexo d -g --config _config.yml,configs/next/config.yml
```


如果
- 出现 The "mode" argument must be integer. 报错
- 生成的 public 目录下所有文件内容均为空
可能是 node 版本过高导致，切换下再执行就好

```shell
$ n 13.14.0
```
