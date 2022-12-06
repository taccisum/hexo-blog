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


## Save resume as PDF

At first, Open Chrome broswer console(via F12) and remove the unnecessary elements by the code below

```js
$('.header').remove()
$('.back-to-top').remove()
$('#valine-comments').remove()
$('.footer').remove()
```

And then, use browser's print function(command-p) to save resume page as PDF file.
