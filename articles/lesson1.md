看过了前面的简介，现在我们来开始写我们的第一个webGL的应用，来使用最少的可以执行的代码来绘制一个三角形。
在开始介绍之前我们先对今天所要写的东西有一个整体的认知。

![lesson1](https://cloud.githubusercontent.com/assets/4397546/10361166/d95447c6-6dd8-11e5-9971-2067211bfa15.png)

webGL其实只是对于一些底层渲染和计算函数的封装，在这里我们通过HTML层来展示所绘制的图形，用shader来进行真正的渲染，shader是webGL的一部分，
作用在GPU之上，来进行渲染工作。

### 骨架
写过前端的同学都知道，在前端的架构中，HTML是骨架，CSS是血肉，为什么这么说呢？因为HTML定义了文档的内容，CSS丰富了各个元素的位置，在本节之中我们不必过多的去考虑CSS给整个文档带来的变化，我们只是使用HTML这个容器来进行图形的绘制。

所以在这里我们先来定义一下如何在浏览器中使用webGL

```html
<html>
<head>
</head>
<body onload="webGLStart()">
<canvas	id='webgl-container'>
</canvas>
</body>
</html>	

```

典型的HTML结构，定义了head、body等等，在这里需要注意的是我们定义了一个canvas作为webGL的容器，
当页面加载完成时执行`webGLStart()`函数开始对画板进行渲染。

之后对于webgl的行为我们使用javascript来进行描述

## 行为

接下来我们来开始进入神奇的webgl的世界，首先我们实现之前定义过的`webGLStart`函数，这个函数是整个webGL的入口

```js
function webGLStart(){
	var canvas = document.querySelector('#webgl-container'); //获取canvas的DOM节点
	initGL(canvas); //初始化WebGL
	initShaders(); //初始化Shader
	initBuffers(); //初始化Buffer
	gl.clearColor(0.0,0.0,0.0,1.0); //清除页面颜色
	drawScene(); //绘制图形
}

```

#### initGL

想象一下在实际的生活中，我们准备去画画的时候需要哪些因素？

> 人，画笔，画板


okokok，在这里canvas就相当于你手中的纸，`initGL`的功能就是从一堆纸(canvas)中挑出(select element)你需要使用的纸，
然后把它装在画板中准备使用。


```js
function initGL(canvas){
	try{
		gl = canvas.getContext("webgl") || canvas.getContext("experimainal-webgl");
		gl.viewportWidth =  canvas.width;
		gl.viewportHeight = canvas.height;
	}catch(e){
		//error handlering...
	}
	if(!gl){
		alert("error in initGL");
	}
}

```

#### initShader

然后我们再来想想接下来该如何去做，纸准备好了当然还需要笔，所以在initShaders这个函数中，我们用来定义“画笔”;
Shader的意思其实是我们去告诉计算机在哪儿去画（渲）图（染）。下面的代码展示了最简单的shader：

```html

<script id='shader-vs' type='x-shader/x-vertex'>
	attribute vec3 aVertexPosition;
	void main(void){
		gl_Position = vec4(aVertexPosition,1.0);
	}
</script>

<script id='shader-fs' type='x-shader/x-fragment'>
	precision mediump float;
	void main(void){
		gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
	}
</script>


```
WOW,之前的你肯定只在`script`标签中写过JavaScript。
这长得好像C语言的东西语法规则是什么鬼呢？
其实这就是Graphics Language Shader Language(GLSL)，我们通过在这里定义的shader，实现与GPU的交互。

在`fragment shader`中我们定义了一个[precision mediump float;](http://stackoverflow.com/questions/13780609/what-does-precision-mediump-float-mean),这一句实际上是定义了GPU处理浮点数的精度，暂且不谈。
在下面的main函数中，我们将颜色定义为`vec4(r,g,b,a)`的值。其中gl_FragColor 是一个内置变量，代表了颜色数据。

> vec4 是有四个参数的四维向量在这里代表了RGBA四个颜色通道

在`vertex shader`中我们定义了类型为`vec3`的`attribute`变量`aVertexPosition` ，表示三维坐标点的位置，之后我们通过vec4这个构造函数将三维坐标点转化为四维。
然后将所得到的点得坐标赋值给内置变量gl_Position来得到图形在空间中的位置。

所以在initShader中我们将如上所写的shader编译到我们的上下文中。


```js
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
	}

```
这个函数做了以下几件事：
* 利用`getShader`函数获得到了编译后的`fragmentShader(fs)`和`vertexShader(vs)`
* 将`shader`传入gl作用域，然后编译了一个`program`来处理webGL
* 将所需要使用的`aVertexPosition`的索引值(ID)传递给program

`Program`又是什么东西？
program可以说就是你使用fragmentShdaer和vertexShader所编译出的上下文作用域，当有不同的fragmentshader和vertexshader时，可以建立不同的shader来分别处理他们。

`getUniformLocation`这个函数用来做什么？
我们通过传入fs和vs来得到一个program之后，program就包含了在fs和vs中定义的变量的索引，getUnifromLocation就是获取uniform类型的变量的索引，方便在webgl函数中将数据传递到glsl中去。

#### getShader

其实在（几乎）所有的webGL程序中我们需要自己去写vertex-shader和fragment-shader。在今天这个例子我们这样来写：

接下来我们开始编写getShader函数，这个函数是一个工具函数，将script中所定义的shader编译成真正的“shader”

```js
function getShader(gl,id){
	var shaderScript = ducument.getElementById(id);
	if(!shaderScript){return null;}
	var str = "";
	var k = shaderScript.fristChild;
	while(k){
		if(k.nodeType == 3){ //TEXT_NODE
			str += k.textContent;
		}
		k = k.nextSibling;
	}
	var shader;
	if (shaderScript.type == "x-shader/x-fragment"){
		shader = gl.createShader(gl.FRAGMENT_SHADER);
	}else if(shaderScript.type == "x-shader/x-vertex"){
		shader = gl.createShader(gl.VERTEX_SHADER);
	}else{
		return null;
	}
	gl.shaderSource(shader,str);
	gl.compileShader(shader);
	if(!gl.getShaderParameter(shader,gl.COMPILE_STATUS)){
		alert(gl.getShaderInfoLog(shader));
		return null;
	}
	return shader;
}
```
这个函数主要做了两件事：

1. 获取到shader的dom，将所有的shader中的文本提取出来
2. 编译shader并返回

#### initBuffers

画笔和画板准备好之后我们就要开始编写逻辑了。
怎么画？画什么？就都是在这部分完成的。

首先定义保存三角形坐标点的全局变量：
```js
var triangleVertexPositionBuffer; //用来保存三角形顶点数据

function initBuffers(){
	triangleVertexPositionBuffer = gl.createBuffer();
	gl.bindBuffer(gl.ARRAY_BUFFER,triangleVertexPositionBuffer);
	var vertices = [
		 0.0 , 1.0, 0.0, // point 1 (x,y,z)
		-1.0 ,-1.0, 0.0,//  point 2 (x,y,z)
		 1.0 ,-1.0, 0.0 //  point 3 (x,y,z)
 	];
 	gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices),gl.STATIC_DRAW);
 	triangleVertexPositionBuffer.itemSize = 3;
	triangleVertexPositionBuffer.numItems = 3;
}
```
在这段代码里，我们创建了一段buffer用来存储三角形的顶点数据，
并且将这个buffer绑定到`gl.ARRAY_BUFFER`之中去。
**`gl.ARRAY_BUFFER`**是[什么鬼](http://stackoverflow.com/questions/14802854/what-does-the-target-gl-array-buffer-mean-in-glbindbuffer)呢？这其实GLSL只是在内部定义的一个地址，将向量数据绑定到这个地址中去使用。
接下来我们定义了一个矩阵的数组用来存储我们三角形的顶点坐标。在WebGL中，我们使用`[-1,1]`的坐标点来描绘整个象限。

>注意这个矩阵形式和数学中得矩阵表示方法并不一样，比如在数学中，我们有如下的矩阵
> ```
> |  0.0, 1.0, 2.0  |
> |  3.0, 4.0, 5.0  |
> |  6.0, 7.0, 8.0  |
> 但是在WebGL的世界我们要表示如上矩阵，就得写成这样
> |  0.0, 3.0, 6.0  |
> |  1.0, 4.0, 7.0  |
> |  2.0, 5.0, 8.0  |
> ```

bufferdata这个函数将vertices的数据绑定到gl.ARRAY_BUFFER中。
而在最后两行，通过设置buffer的itemSize和numItems表示出三维(x,y,z)和点的数量(x1,x2,x3);


## drawScene 

这时候在我们的代码中还剩下最重要的一步，就是将之前所讲的内容画到canvas中去。
drawScene函数闪亮登场！

在这里我们会着重介绍webGL绘制的过程：
* 首先定义上下文作用域(gl)的视窗，也就是说当前画板的大小
* 将在这个gl上的内容都给clear掉
* 之后向buffer中绑定三角形的数据,并格式化
* 并且将它画到画板中去。

```js
function drawScene(){
	gl.viewport(0, 0, gl.viewportWidth, gl.viewportHeight);
	gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
	gl.bindBuffer(gl.ARRAY_BUFFER,triangleVertexPositionBuffer);
	gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute,triangleVertexPositionBuffer.itemSize,gl.FLOAT,false,0,0);
	gl.drawArrays(gl.TRIANGLES,0,triangleVertexPositionBuffer.numItems);
}
```

至此我们的三角形就算是画完了。完整的代码可以戳[这里](
https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/1.html)

有了以上的介绍，相信你已经对于webGL有了最基本的了解，下面留一个小作业：

> 如何在WebGL中绘制一个正方形呢？ 长方形呢？

试试做一下咯。之后会介绍更多关于webgl的知识，不要走开哟。
