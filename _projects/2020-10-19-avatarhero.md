---
title: 'AvatarHero'
subtitle: 'Create your own avatar or gaming logo'
date: 2020-10-18 00:00:00
featured_image: '/images/AH.png'
excerpt: 'A dynamic avatar builder MVP built using the following technologies: Vue.js and Canvas for the front end. Node.js, Node-canvas, DynamoDB and Paypal integration in the API.'
---

![](/images/AH.png)

A dynamic avatar builder MVP built using the following technologies: Vue.js and Canvas for the front-end. Node.js, node-canvas, DynamoDB and Paypal integration in the API. 

![](/images/53844d39-a105-4739-8941-2f64d369f878/512.png)

### Rendering

The avatar rendering is written in Vue.js and Canvas, each layer is rendered and animated individually allowing smooth animations and transitions when customising your avatar. 

When an avatar is purchased and verified with Paypal the a simplified rendering is performed on the server without the watermark and with higher resolution assets. The assets are then resized, zipped and uploaded to S3. The order is recorded in DynamoDB and confirmation is emailed to the user. 

```js
module.exports = {
  build: async (avatarString, text) => {
    let avatar = codeParser(avatarString);
    let canvas = createCanvas(1024, 1024);
    let context = canvas.getContext('2d');

    let exclusions = calculateExclusions(avatar);

    await loadAndRenderImage(context, avatar.flair, exclusions);
    await loadAndRenderImage(context, avatar.background, exclusions);
    drawText(context, avatar.font, text.trim().toUpperCase());
    await loadAndRenderImage(context, avatar.variant, exclusions);
    await loadAndRenderImage(context, avatar.hair, exclusions);
    await loadAndRenderImage(context, avatar.mouth, exclusions);
    await loadAndRenderImage(context, avatar.extras, exclusions);
    await loadAndRenderImage(context, avatar.eyewear, exclusions);
    await loadAndRenderImage(context, avatar.headgear, exclusions);

    return [
      { image: canvas.createPNGStream(), name: '1024.png' },
      { image: resize(canvas, 800), name: '800.png' },
      { image: resize(canvas, 512), name: '512.png' },
      { image: resize(canvas, 400), name: '400.png' },
      { image: resize(canvas, 360), name: '360.png' },
      { image: resize(canvas, 256), name: '256.png' },
      { image: resize(canvas, 128), name: '128.png' },
    ];
  }
}
```

### Font Styling

Font styling in Canvas can be an interesting challenge, the font effect is achieved by rendering the font multiple times at different sizes. The text is also constrained to 900 width using `context.measureText`.

```js
let outline = function(context, size, fontData) {
    let { w, h } = size;
    let { family, weight } = fontData;

    var fontSize = 150;
    
    context.globalAlpha = this.opacity;
    context.font = '${weight} ${fontSize}px ${family}';
    context.textAlign = "center"; 

    var textSize = context.measureText(this.asset);
    while (textSize.width > 900) {
        fontSize -= 2;
        context.font = '${weight} ${fontSize}px ${family}';
        textSize = context.measureText(this.asset);
    }

    let x = w/2 + this.position.x;
    let y = h/2 + 400 + this.position.y - ((150-fontSize) /2);

    context.shadowColor = "black";
    context.shadowBlur = 20;
    context.lineCap = 'round';
    context.lineJoin = 'round';

    // outer
    context.strokeStyle = "#878787";
    context.lineWidth = 46;
    context.strokeText(this.asset, x, y+3);

    context.shadowBlur = 0;

    // inner
    context.strokeStyle = "#000000";
    context.lineWidth = 26;
    context.strokeText(this.asset, x, y+3);

    // fake 3d
    context.fillStyle = "#959595";
    context.fillText(this.asset, x, y+6); 

    // font
    context.fillStyle = "#FFFFFF";
    context.fillText(this.asset, x, y); 

    context.globalAlpha = 1;
}

export default {
    outline
}
```
