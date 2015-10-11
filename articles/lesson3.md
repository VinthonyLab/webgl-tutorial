在之前的介绍之中，我们定义了一个`[-1,1]`的三维空间来表示我们需要绘制的空间。
但是如果我们绘制一个复杂的图形的话，所有的点我们不可能都定义在`[-1,1]`之间，这样的话我们需要用小数来表示点得位置
会有一系列的精度问题等。
所以今天我们介绍几个概念来对我们的图形进行一系列的变换，让它“正确的”显示在屏幕之中。

### 平移

首先我们先来介绍一些数学知识：
假设我们想在这个空间中去平移一个元素，原物体中的一个坐标点(x,y,z) 将会变成 (x+t,y+t,z+t)
所以在这里我们希望将坐标点表示成一个向量，通过一个转换矩阵来实现这个过程。
首先我们以单位矩阵开始一步步进行变换，我们这样来进行单位矩阵和坐标点之间的作用。

```
| 1  0  0 |       | x |     | x |
| 0  1  0 |   *   | y |  =  | y |
| 0  0  1 |       | z |     | z |
```

当我们需要进行t的转换时:

```
| ?  ?  ? |       | x |     | x+t |
| ?  ?  ? |   *   | y |  =  | y+t |
| ?  ?  ? |       | z |     | z+t |
```

在三维矩阵运算中，我们很难去定义这样一个矩阵满足如上的内容。
所以我们通过增加一维内容在实现t的转换：

```
| 1  0  0  t |       | x |      | x+t |
| 0  1  0  t |       | y |      | y+t | 
| 0  0  1  t |   *   | z |   =  | z+t |
| 0  0  0  1 |       | 1 |      |  1  |
```
> 在shader中，gl_Position 的值为四维就是因为这个原因。可以通过四维矩阵乘法来保证齐次性变换


### 旋转 

在webGL中，旋转操作对于某个物体来说是作用于这个物体上的每个顶点相对于原点沿着某条特定的线的旋转距离。
特殊情况下为绕X，Y，Z轴，在下面的情况中我们固定z轴，得到：

![qq20151011-1](https://cloud.githubusercontent.com/assets/4397546/10416104/a97576ca-7039-11e5-924e-56d16a0ea443.png)

故对于坐标点(x0,y0,z0)和目标坐标点(xs,ys,zs),如果转动了`Θ`角度：

![eq1](http://www.sciweavers.org/tex2img.php?eq=%20%20%5Cfrac%7Bx0%7D%7B%5Csqrt%7B%20x0%5E%7B2%7D%2B%20y0%5E%7B2%7D%20%7D%20%7D%20%3D%20cos%20%5Calpha%20&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0)

![eq2](http://www.sciweavers.org/tex2img.php?eq=%20xs%20%3D%20cos%28%20%5Calpha%20%2B%20%5Ctheta%20%29%20%2A%20%20%5Csqrt%7B%20x0%5E%7B2%7D%20%2B%20%20y0%5E%7B2%7D%20%7D%20&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0)

根据三角形倍角公式得到：

![eq3](http://www.sciweavers.org/tex2img.php?eq=cos%28%20%5Calpha%20%2B%20%20%5Ctheta%20%29%20%3D%20cos%20%20%5Calpha%20cos%20%5Ctheta%20%20-%20sin%20%5Calpha%20sin%20%5Ctheta%20&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0)

综上可以得到：

`cosΘx0 - sinΘy0 = xt`
`cosΘy0 + sinΘx0 = yt`

```

| cosΘ  -sinΘ  0  0 |       | x |      | cosΘx0 - sinΘy0 |
| sinΘ   cosΘ  0  0 |       | y |      | sinΘx0 + cosΘy0 | 
|    0      0  1  0 |   *   | z |   =  |  z  |
|    0      0  0  1 |       | 1 |      |  1  |

```
同理，固定x轴或者y轴可以通过同样的方法计算出来转换矩阵。

### 缩放

缩放和扩放过程就是对每个维度的坐标值乘以一定的系数得到，故：

```
xt = x0 * w;
yt = y0 * w;
zt = z0 * w;
```
所以可以得转换矩阵:


```
| w  0  0  0 |       | x |      | x * w |
| 0  w  0  0 |       | y |      | y * w | 
| 0  0  w  0 |   *   | z |   =  | z * w |
| 0  0  0  1 |       | 1 |      |   1   |
```

### 代码

在webGL中使用如上变换的方式有两种，一种是在javascript阶段去使用还有一种是在shader阶段，
由于shader阶段是在gpu中进行运算，而且GLSL对于矩阵乘法有很好的优化，所以我们选择在shader中进行运算。

首先我们通过`gl.getUniformLocation`来得到uniform类型变量的索引：

```js
var sMatrix = [.....];
gl.getUniformLocation(shaderProgram,"uMVMatrix");
```
然后再将我们转换矩阵的坐标点传入到vertexShader中去：

```
gl.uniformMatrix4fv(shaderProgram.uMVMatrixUniform,false,new Float32Array(sMatrix));
```

在shader中，我们这样处理：
```
	uniform mat4 uMVMatrix;
	void main(void) {
		gl_Position = uMVMatrix * vec4(aVertexPosition, 1.0);
		vColor = aVertexColor;
	}
```
因为在glsl中所有的运算符已经重载过，所以可以直接进行矩阵乘法的运算。
当进行多个变幻时，在这里矩阵按照左乘法则，先计算左边和坐标向量的乘积，再进行下一步运算。

故这里，假设我们有`tMatrix`,`sMatrix`,`rMatrix`三个矩阵来分别表示平移，缩放，旋转。
进行乘法的顺序应该是：

```
gl_Position = tMatrix * rMatrix * sMatrix * originalVector;
```
先平移的话会导致整个过程错误。


在这里有四个例子来表示刚才所说的内容：

1. [平移](https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/3-1.html)
2. [旋转](https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/3-2.html)
3. [缩放](https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/3-3.html)
4. [结合](https://github.com/VinthonyLab/webgl-tutorial/blob/master/examples/3-4.html)

