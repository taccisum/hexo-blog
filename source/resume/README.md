
## Save resume as PDF

At first, Open Chrome broswer console(via F12) and remove the unnecessary elements by the code below

```js
$('.header').remove()
$('.back-to-top').remove()
$('#valine-comments').remove()
$('.footer').remove()
$('.sidebar-toggle').click()
$('.sidebar-toggle').remove()
```

And then, use browser's print function(command-p) to save resume page as PDF file.