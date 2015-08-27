在之前的介绍中我们介绍了怎样去构建一个WebGL版本的`helloworld`.画出了一个三角形和一个正方形。
在本节内容之中我们将着重介绍如何去给这两个图形上色。

我们只需要更改很少的代码就能够在之前的基础之上完成这这部分内容。

在修改代码之前我们先来了解一下WebGL是如何来渲染的。
如图所示：

![the process of vertex](http://learningwebgl.com/lessons/lesson02/simple-rendering-pipeline.png)

首先在我们javascript的代码中，我们定义的变量在`drawScene`函数中使用`vertexAttribPointer`这个函数将vertex数据传递到shaderProgram的attribute类型变量和将uniform类型的变量传递到vertex-shader中。vertex-shader中有一个必须的变量--`gl_Position`,表明了包含的定点的坐标，当vertex-shader完成之后，`webGL`程序调用fragment-shader，fragment-shader会逐个像素点来渲染图像。所以有些时候我们将fragment-shader称作`pixel-shader`。在fragment-shader中，我们必须去定义`gl_FragColor`，它为varying类型的变量，表示的是渲染像素点的**背景**颜色。之后，当我们进行完fragment-shader，所有的数据将会放入`frame buffer`，这才是之后我们会在屏幕上看到的内容，我们将在后面继续介绍它。

所以在这里我们先来看看相比于第一节的代码，本节的代码又有哪些改变呢？

首先来看看shader：

```html

<script id="shader-fs" type="x-shader/x-fragment">
  precision mediump float;

  varying vec4 vColor;

  void main(void) {
    gl_FragColor = vColor;
  }
</script>

<script id="shader-vs" type="x-shader/x-vertex">
  attribute vec3 aVertexPosition;
  attribute vec4 aVertexColor;

  uniform mat4 uMVMatrix;
  uniform mat4 uPMatrix;

  varying vec4 vColor;

  void main(void) {
    gl_Position = uPMatrix * uMVMatrix * vec4(aVertexPosition, 1.0);
    	vColor = aVertexColor;
  }
</script>

```
首先在shader部分，我们在vertex-shader中增加了一个arribute变量`aVertexColor（vec4类型，rgba）`,并且定义了一个varying类型的vColor属性，用于向`fragment shader`传递数据（由前图可知，首先进行的是vertex-shader程序，之后是fragment-shader程序，varying类型的变量可以在vertex和fragment中传递内容），同时，我们在fragment中定义了这个vColor属性，并且将它从vertex shader中得到的值传递给`gl_FragColor`,完成赋值操作。

道理讲完，可是真正去实现上色的部分在哪儿呢？
首先我们在`initShaders`函数中，给我们的shaderProgram增加一个属性`vertexColorAttribute`,这能干嘛呢，其实就是获取到了aVertexColor的地址，便于将数据存入，所以现在我们的代码变成了这个样子：

> 注释部分为新加入

```js
var shaderProgram;
	function initShaders(){
		
		var fragmentShader = getShader(gl,"shader-fs");
		var vertexShader = getShader(gl,"shader-vs");

		shaderProgram = gl.createProgram();
		gl.attachShader(shaderProgram, vertexShader);
		gl.attachShader(shaderProgram, fragmentShader);
		gl.linkProgram(shaderProgram);

		if (!gl.getProgramParameter(shaderProgram, gl.LINK_STATUS)){
			alert("couldn't init shaders")
		}
		gl.useProgram(shaderProgram);
		shaderProgram.vertexPositionAttribute = gl.getAttribLocation(shaderProgram, "aVertexPosition");	
		gl.enableVertexAttribArray(shaderProgram.vertexPositionAttribute);

		//shaderProgram.vertexColorAttribute = gl.	getAttribLocation(shaderProgram,"aVertexColor");
		//gl.enableVertexAttribArray(shaderProgram.vertexColorAttribute);	

		shaderProgram.pMatrixUniform = gl.getUniformLocation(shaderProgram,"uPMatrix");
		shaderProgram.mvMatrixUniform = gl.getUniformLocation(shaderProgram,"uMVMatrix");
	}

```

接下来就是讲数据填入aVertexColor这个变量中了，这个过程在`initBuffers`这个过程中进行。
类似于形状的绘制，我们先定义了一个定点颜色的矩阵，然后将这个矩阵的数据绑定到`ARRAY_BUFFER`中，并且定义了每个点的维度和个数。

```js
var colors = [
		1.0,0.0,0.0,1.0,
		0.0,1.0,0.0,1.0,
		0.0,0.0,1.0,1.0
	];
	gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(colors), gl.STATIC_DRAW);
	triangleVertexColorBuffer.itemSize = 4;
	triangleVertexColorBuffer.numItems = 3;

```
之后在drawScene中,通过vertexAttribPointer将triangleVertexColorBuffer中的数据通过ARRAY_BUFFER传递到shaderProgram.vertexColorAttribute中。

```js
gl.bindBuffer(gl.ARRAY_BUFFER,triangleVertexColorBuffer);
gl.vertexAttribPointer(shaderProgram.vertexColorAttribute,triangleVertexColorBuffer.itemSize,gl.FLOAT,false,0,0);
```

到这里，整个流程已经跑通，我们的颜色就已经上好啦。

完整的例子戳[这里](https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/2.html)
