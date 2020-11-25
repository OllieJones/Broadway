## Decoded video sizing and clipping

### Introduction

A video source (an H.264 data stream) has a particular size, a particular width and height, that's dependent on the camera and scaling by the video encoder.  That size may not match the size of the element where you want to show the video to the viewer.

So, how can we handle that situation? Several ways.

* Stretch the video *anamorphically*. That is, give it different scales in width and height so it fill the element. This distorts the video, so stretching is usually not a good choice.

* Scale the video, by *letterboxing* it so the whole video can be seen in the element. If the element (the div) for showing the video is too high for the video we get this kind of result, with blank areas at the top and bottom of the element. When rendering the video we stretch it so it fits the width of the div.

```
+------------------------------------------+
|                                          |
|                                          |
|VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV|
|VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV|
|VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV|
|VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV|
|VVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVVV|
|                                          |
|                                          |
+------------------------------------------+
```

   If the element for showing the div is too wide for the video we get this kind of result, with blank areas at the left and right of the video. When rendering the video we stretch it so it fits the height of the div.

```
+------------------------------------------+
|      VVVVVVVVVVVVVVVVVVVVVVVVVVVVVV      |
|      VVVVVVVVVVVVVVVVVVVVVVVVVVVVVV      |
|      VVVVVVVVVVVVVVVVVVVVVVVVVVVVVV      |
|      VVVVVVVVVVVVVVVVVVVVVVVVVVVVVV      |
+------------------------------------------+
```
 
 * *Clip* the video, trimming off part of it on the left and right, or top and bottom, so it fills the element. This gives a pleasing result to the viewer, but loses a bit of it.  For many applications clipping is the right choice. 
 
 ```
    +-----------------------------+
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
....|VVVVVVVVVVVVVVVVVVVVVVVVVVVVV|....
    +-----------------------------+
```

### HTML and CSS

We need a div for Broadway to use to render the incoming video.  For our letterboxing or clipping to work we need two nested divs with HTML like this.  Broadway creates its own canvas element in the div.videocontainer.

```html
<div id="container">
  <div class="videocontainer"></div>
</div>
```

We need CSS like this

```css
    body {
      padding: 0;
      margin: 0;
      overflow: hidden;
    }

    div#container div.videocontainer {
      width: 100vw;
      height: 100vh;
      background-color: transparent;
      display: block;
      object-fit: fill;
      position: relative;
    }

    div#container div.videocontainer > canvas {
      padding: 0;
      margin: auto;
      display: block;
      position: absolute;
      background-color: transparent;
      top: 0;
      bottom: 0;
      left: 0;
      right: 0;
    }

    div#container div.videocontainer.scale-to-fit > canvas {
      width: 100%;
      height: 100%;
    }

```

### Incoming video dimensions

We need to handle Broadway's `onFrameSizeChange` event. When delivered, that event tells us some important dimensions.  When we get the incoming video dimensions, or if they  change, we need to resize the element -- we'll call it the *window* -- where we render the video.

```javascript
const videoTag = document.querySelector('div#container div.videocontainer')
let broadwayPlayer = // create the BroadwayPlayer object here.
videoTag.appendChild(broadwayPlayer.domNode)
let sourceWidth = 0
let sourceHeight = 0
broadwayPlayer.onFrameSizeChange = function( frameData ) {
  broadwayFrame = frameData
  if (    sourceWidth !== frameData.sourceWidth 
       || sourceHeight !== frameData.sourceHeight) {
    sourceWidth = frameData.sourceWidth
    sourceHeight = frameData.sourceHeight
    console.log ("incomding video %ix%i", sourceWidth, sourceHeight)
    resizeWindow()
   }
}
```
We'll find the dimensions we need in `frameData.sourceWidth` and `frameData.sourceHeight` when this event fires. But, it doesn't fire until Broadway has decoded its first frame on a new video stream.

And we need to respond to the dimension change. To do that we call the `resizeWindow()` function. 

### Resizing the window

`resizeWinod()` looks something like this. It has to do some work to handle the letterboxing or clipping. 

```javascript
const scalingMethod = 'scale'      // or 'letterbox' or 'stretch' 
let resizeTimeout

function resizeWindow() {
  resizeTimeout = null
  const canvasTag = videoTag.querySelector( 'canvas' )
  if( !canvasTag ) return
  const canvasStyle = canvasTag.style
  const rect = videoTag.getBoundingClientRect()
  var tWidth = rect.width
  var tHeight = rect.height
  var offset = 0
  var dAspect
  var vAspect
  if( scalingMethod !== 'stretch') {
    if( videoHeight <= 0 || rect.height <= 0 ) {
      /* bogus height values, don't do anything, yet */
      return
    }
    dAspect = decodedWidth / decodedHeight
    vAspect = rect.width / rect.height
    if( scalingMethod === 'clip' ) {
      /* scale-clip */
      if( dAspect > vAspect ) {
        /* decoded video aspect wider than viewport aspect, clip sides */
        tWidth = tHeight * dAspect
        offset = Math.round( (rect.width - tWidth) * 0.5 )
        canvasStyle.top = 0 + 'px'
        canvasStyle.left = offset + 'px'
      }
      else {
        /* decoded video aspect narrower than viewport aspect, clip top and bottom */
        tHeight = tWidth / dAspect
        offset = Math.round( (rect.height - tHeight) * 0.5 )
        canvasStyle.left = 0 + 'px'
        canvasStyle.top = offset + 'px'
      }
    }
  }
  const dimensions = {
        width:Math.round( tWidth ), 
        height:  Math.round( tHeight )}
  broadwayPLayer.setTargetDimensions( dimensions )
}
```





