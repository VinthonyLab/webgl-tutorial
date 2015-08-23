看过了前面的简介，现在我们来开始写我们的第一个webGL的应用。
### 骨架
写过前端的同学都知道，在前端的架构中，HTML是骨架，CSS是血肉，为什么这么说呢？因为HTML定义了文档的内容，CSS丰富了各个元素的位置。
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
当页面加载完成时执行`webGLStart()`函数来绘制图形。

之后对于webgl的行为我们在`script`标签中进行描述。

## 行为

接下来我们来开始进入神奇的webgl的世界，首先我们实现之前定义过得`webGLStart`

```js
function webGLStart(){
	var canvas = document.querySelector('#webgl-container');
	initGL(canvas);
	initShaders();
	initBuffers();

	gl.clearColor(0.0,0.0,0.0,1.0);
	gl.enable(gl.DEPTH_TEST);

	drawScene();
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

#### initShader

```
然后我们再来想想接下来该如何去做，纸准备好了当然还需要笔，所以在initShaders这个函数中，我们用来定义“画笔”:
shader的意思其实是我们所画的图像在执行渲染任务时使用的指令。接下来我们一步步来探索如何编写这个函数：

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
		shaderProgram.pMatrixUniform = gl.getUniformLocation(shaderProgram,"uPMatrix");
		shaderProgram.mvMatrixUniform = gl.getUniformLocation(shaderProgram,"uMVMatrix");
	}

```
在这个函数中我们首先从`shader-fs`和`shader-vs`中得到fragmentShader和vertexShader，
这两个shader是什么东西呢？
> [stackoverflow.com](http://stackoverflow.com/questions/4421261/vertex-shader-vs-fragment-shader)
我们知道我们的物体有点和面，fragmentShader对应的就是面的渲染器，能够实现某个面的特殊的效果，而vertexShader则更多的是关于顶点着色的
之后在shader中，我们创建了一个program，并且将所获得的shader连接到program中，接下来的getUniformLocation为或得这个shaderProgram中
这两个变量`uPMatrix`和`uMVMatrix`的位置，并且赋值到shaderProgram中，为了方便JS去调用，在这里JavaScript的语言特性就比较方便，尽管没有这些属性，还是能够直接使用shaderProgram.XXXX来进行赋值。
而`aVertexPositon`，`uPMatrix`，`uMVMatrix`是从哪儿来的呢？我们在getShader中寻找它的奥秘。

#### getShader

其实在所有的webGL程序中我们需要自己去写vertex-shader和fragment-shader。在今天这个例子我们这样来写：

```html
<script id='shader-fs' type='x-shader/x-fragment'>
	precision mediump float;
	void main(void){
		gl_FragColor = vec4(1.0, 1.0, 1.0, 1.0);
	}
</script>
<script id='shader-vs' type='x-shader/x-vertex'>
	attribute vec3 aVertexPosition;

	uniform mat4 uMVMatrix;
	uniform mat4 uPMatrix;

	void main(void){
		gl_Position = uMVMatrix * uPMatrix * vec4(aVertexPosition,1.0);
	}
</script>

```
WOW,之前的你肯定只在`script`标签中写过JavaScript。
这长得好像C语言的东西语法规则是什么鬼呢？
其实这就是Graphics Language Shader Language(GLSL)，我们通过在这里定义的shader，实现与GPU的交互。

在`fragment shader`中我们定义了一个[precision mediump float;](http://stackoverflow.com/questions/13780609/what-does-precision-mediump-float-mean),这一句实际上是定义了GPU处理浮点数的精度，暂且不谈。
在下面的main函数中，我们将整个材质的颜色定义为`vec4(x,y,z,t)`的值。

> vec4 是有四个参数的四维向量，在这里我们使用单位向量去定义

在`vertex shader`中我们定义了类型为`vec3`的`attribute`变量`aVertexPosition` 在之前的`initShader`函数中，我们就是使用了`getAttribLocation`来得到这个变量的数据。

同理我们定义了类型为`mat4`的`uniform`变量`uMVMatrix`和`uPMatrix`，同样使用`getUniformLocation`来获得这些数据,为了使用这两个变量。我们还需要定义如下的函数：
```js
var mvMatrix = mat4.create();
var pMatrix = mat4.create();
function setMatrixUniforms(){
	gl.uniformMatrix4fv(shaderProgram.pMatrixUniform,false,pMatrix);
	gl.uniformMatrix4fv(shaderProgram.mvMatrixUniform,false,mvMatrix);
}

```
这个函数初始化了这两个矩阵变量，并且将它指向所在的location

> mat4 vs vec3
> 这里涉及到了glsl的数据类型
> matn 表示`n*n`的矩阵
> vecn 表示n维向量



> `uniform` vs. `attribute`
> uniform [vertex/fragment]类似于C语言中的常量，shader程序只能读取，但是不能做修改,用来表示变换矩阵，材质，光照参数和颜色等信息
> attribute [vertex] 这种类型的变量只能存在于vertex shader中，一般用来表示顶点的数据
> PS: varying 变量，该种变量是vertex和fragment之间传递信息使用，一般是vertex修改该种数据的值，然后fragment使用它。所以声明的类别必须一致

shader languange介绍的差不多了，接下来我们开始编写getShader函数，这个函数的功能是将script中所定义的shader编译成真正的“shader”

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

在这个例子里我们先画一个三角形再画一个正方形。

首先定义两个全局变量：
```js
var triangleVertexPositionBuffer; //用来保存三角形顶点数据
var squareVertexPositonBuffer; //用来保存正方形定点数据

function initBuffers(){
	triangleVertexPositionBuffer = gl.createBuffer();
	gl.bindBuffer(gl.ARRAY_BUFFER,triangleVertexPositionBuffer);
	var vertices = [
		 0.0 , 1.0, 0.0,
		-1.0 ,-1.0, 0.0,
		 1.0 ,-1.0, 0.0
 	];
 	gl.bufferData(gl.ARRAY_BUFFER,new Float32Array(vertices),gl.STATIC_DRAW);
 	triangleVertexPositionBuffer.itemSize = 3;
	triangleVertexPositionBuffer.numItems = 3;
}
```
在这段代码里，我们创建了一段buffer用来存储三角形的顶点数据，
并且将这个buffer绑定到`gl.ARRAY_BUFFER`之中去。
**`gl.ARRAY_BUFFER`**是[什么鬼](http://stackoverflow.com/questions/14802854/what-does-the-target-gl-array-buffer-mean-in-glbindbuffer)呢？
这其实GLSL只是在内部定义的一个地址，将向量数据绑定到这个地址中去使用。
接下来我们定义了一个矩阵的数组
```
x1,y1,z1,
x2,y2,z2,
x3,y3,z3

```
bufferdata这个函数将vertices的数据绑定到gl.ARRAY_BUFFER中。
而在最后两行，通过设置buffer的itemSize和numItems表示出三维(x,y,z)和点的数量(x1,x2,x3);

画正方形的代码与之类似，在这里就不展开讲了，直接上代码：

```js
		squareVertexPositionBuffer = gl.createBuffer();
		gl.bindBuffer(gl.ARRAY_BUFFER, squareVertexPositionBuffer);
		vertices = [
			1.0,1.0,0.0,
			-1.0,1.0,0.0,
			1.0,-1.0,0.0,
			-1.0,-1.0,0.0
		]; //变成了四个点
		gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertices), gl.STATIC_DRAW);
		squareVertexPositionBuffer.itemSize = 3;
		squareVertexPositionBuffer.numItems = 4;//四个点
```


## drawScene 

这时候在我们的代码中还剩下最重要的一步，就是将之前所讲的内容画到canvas中去。
drawScene函数闪亮登场！

在这里我们会着重介绍webGL绘制的过程：
* 首先定义上下文作用域(gl)的视窗，也就是说当前画板的大小
* 将在这个gl上的内容都给clear掉
* 由于之前在`initBuffers`中我们只是定义了图形的样子，并没有提到图形的位置，在这一步中我们使用矩阵的透视方法，第一个参数表示的是fov(field-of-view),第二个参数是radio，表示长宽比例，第三个和第四个为透视可视区间[min,max],最后一个参数表示的是这些数据存入的矩阵.
* `indetify`这个函数将`mvMatrix`变成一个单位矩阵
* `translate`是将第二个参数的向量作用在`mvMatrix`上
* 之后向buffer中绑定三角形的数据,并格式化
* 并且将它画到画板中去。

```js
function drawScene(){
	gl.viewport(0, 0, gl.viewportWidth, gl.viewportHeight);
	gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
	mat4.perspective(45,gl.viewportWidth / gl.viewportHeight, 6.0, 100.0, pMatrix);
	mat4.identity(mvMatrix);
	
	mat4.translate(mvMatrix,[0,0,-20.0]);
	gl.bindBuffer(gl.ARRAY_BUFFER,triangleVertexPositionBuffer);
	gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute,triangleVertexPositionBuffer.itemSize,gl.FLOAT,false,0,0);
	setMatrixUniforms();
	gl.drawArrays(gl.TRIANGLES,0,triangleVertexPositionBuffer.numItems);

	mat4.translate(mvMatrix,[4.0,3.0,-5.0]);
	gl.bindBuffer(gl.ARRAY_BUFFER,squareVertexPositionBuffer);
	gl.vertexAttribPointer(shaderProgram.vertexPositionAttribute,squareVertexPositionBuffer.itemSize,gl.FLOAT,false,0,0);
	setMatrixUniforms();
	gl.drawArrays(gl.TRIANGLE_STRIP,0,squareVertexPositionBuffer.numItems);
}
```

至此我们的三角形和正方形就算是画完了。完整的代码可以戳这里[
https://github.com/VinthonyLab/webgl-tutorial/blob/master/trangle-and-square.md]

之后会介绍更多关于webgl的知识，不要走开哟。
