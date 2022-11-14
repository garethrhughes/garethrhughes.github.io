---
title: 'Base64 image helper for Handlebars'
author: gareth
date: 2021-07-07 18:00:00 +1000
excerpt: Creating a base64 image helper for Handlebars for use with the PDF Generator
categories: [Development]
tags: [development, javascript, aws]
mermaid: true
math: true
---

The PDF generator only currently accepts a HTML payload with no external references, this means that all CSS and images and images need to be embedded in the HTML directly.

We don't want to check those base64 images directly in the templates as that could become difficult to manage. To solve this we created a handler which will convert an image into a base64 encoded string when the templates are compiled.

### Template

{% raw %}
```html
<img src="{{base64ImageSrc './logo.png'}}" alt="image" />
```
{% endraw %}

### Helper

{% raw %}
```javascript
const base64ImageSrc = (imagePath: string) => {
  const bitmap = fs.readFileSync(path.join(__dirname, 'images', imagePath));
  const base64String = new Buffer(bitmap).toString('base64');

  return new Handlebars.SafeString(`data:image/png;base64,${base64String}`);
}

const hbs = exphbs({
  layoutsDir: path.join(__dirname, '/layouts'),
  extname: '.hbs',
  helpers: { 
    base64ImageSrc: base64ImageSrc
  }
});
```
{% endraw %}

In this version we're using express & handlebars-express to wire this up, this will be later converted into .NET 5 however this is a simple POC to test out some of the templating & PDF conversion.