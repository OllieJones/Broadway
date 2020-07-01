# Rough notes about WEBGl flow of control.

Webgl path. How does YUVCanvas.js work, anyhow?

```
YUVCanvas.js:219  canvas.getContext('webgl')

247: yuv420 vertexShaderScript:
attribute vec4 vertexPos;
attribute vec4 texturePos;
attribute vec4 uTexturePos;
attribute vec4 vTexturePos;
varying vec2 textureCoord;
varying vec2 uTextureCoord;
varying vec2 vTextureCoord;
void main()
{
  gl_Position = vertexPos;
  textureCoord = texturePos.xy;
  uTextureCoord = uTexturePos.xy;
  vTextureCoord = vTexturePos.xy;
}

265: fragmentShaderScript:
precision highp float;
varying highp vec2 textureCoord;
varying highp vec2 uTextureCoord;
varying highp vec2 vTextureCoord;
uniform sampler2D ySampler;
uniform sampler2D uSampler;
uniform sampler2D vSampler;
uniform mat4 YUV2RGB;
void main(void) {
  highp float y = texture2D(ySampler,  textureCoord).r;
  highp float u = texture2D(uSampler,  uTextureCoord).r;
  highp float v = texture2D(vSampler,  vTextureCoord).r;
  gl_FragColor = vec4(y, u, v, 1) * YUV2RGB;
}

```



333: YUV2RGB (BT.601)

363: useProgram made of the above two scripts

365:  var YUV2RGBRef = gl.getUniformLocation(program, 'YUV2RGB'); reference to item declared in fragmentShaderScript 

366:   gl.uniformMatrix4fv(YUV2RGBRef, false, YUV2RGB);   Stuff 4x4 matrix into referenced location

378:  var vertexPosBuffer = gl.createBuffer();   // get buffer ... 

379:  gl.bindBuffer(gl.ARRAY_BUFFER, vertexPosBuffer);   /// ... and make it into an array buffer.

380:  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([1, 1,  -1, 1,  1, -1,   -1, -1]), gl.STATIC_DRAW); // triangle strip, static.
        (drawn with textures from the decoded image).

382:  var vertexPosRef = gl.getAttribLocation(program, 'vertexPos');   //ref to item declared in vertexShaderScript

383:  gl.enableVertexAttribArray(vertexPosRef);  // enable it.

384:  gl.vertexAttribPointer(vertexPosRef, 2, gl.FLOAT, false, 0, 0); // item, size, type, normalized, stride, offset.

424-431. Same deal for texturePos, with array [1, 0, 0, 0, 1, 1, 0, 1]

436-455. Same deal for uTexturePos, vTexturePos (chroma)

463: initTextures  for y, u, v

497: initTextures

500: createTexture

501: bindTexture TEXTURE_2D
     (this is a spell to create a simple texture for yuv -> rgb conversion
```
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST); //magnification, point sampling not LINEAR
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);  // minification 
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);  //wrapping in s direction
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);  //wrapping in t direction
```
508: bindTexture to null

470: var ySamplerRef = gl.getUniformLocation(program, 'ySampler'); // in the shader script.

471: gl.uniform1i(ySamplerRef, 0);  // bind ySampler to texture unit 0

476:  // uSampler to unit 1

481:  // vSampler to unit 2

288 render an image.  It's provided in a linear buffer with w x h luma and downsampled chroma, all concatenated.

80: set up to render a picture.

105  gl.viewport(0, 0, width, height);  turns 0-1 /0-1 coordinates into 0-width / 0-height

107 etc: possible scaling stuff.

114, 126, 139.  Bind the texturePosBuffers to [1,0, 0,0, 1,1, 0,1] for dynamic draw.

142:    gl.activeTexture(gl.TEXTURE0);  // select texture unit 0, previously rigged.
        gl.bindTexture(gl.TEXTURE_2D, yTextureRef);   // bind the 2d texture.
        
144:    gl.texImage2D(gl.TEXTURE_2D, 0, gl.LUMINANCE, yDataPerRow, yRowCnt, 0, gl.LUMINANCE, gl.UNSIGNED_BYTE, yData);

Specify the image for the texture.  0 (base level, no mipmap ,luma, w, h, border=0. luma, datatype, data)

Do it all again for the chroma images.

*        gl.activeTexture(gl.TEXTURE1);
*        gl.bindTexture(gl.TEXTURE_2D, uTextureRef);
*        gl.texImage2D(gl.TEXTURE_2D, 0, gl.LUMINANCE, uDataPerRow, uRowCnt, 0, gl.LUMINANCE, gl.UNSIGNED_BYTE, uData);

*        gl.activeTexture(gl.TEXTURE2);
*        gl.bindTexture(gl.TEXTURE_2D, vTextureRef);
*       gl.texImage2D(gl.TEXTURE_2D, 0, gl.LUMINANCE, vDataPerRow, vRowCnt, 0, gl.LUMINANCE, gl.UNSIGNED_BYTE, vData);


 154:   gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4); // draw the first four points in the array as a tristrip, rigged at line 380.

