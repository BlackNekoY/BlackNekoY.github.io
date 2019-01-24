---
title: OpenGL(2) - 着色语言
tags: []
keywords:
  - a
  - b
  - c
comments: true
date: 2017-09-02 17:36:29
updated:
categories:
description:
image:
---
<p class="description"></p>

<img src="" class="img-topic"/>

<br />

# OpenGL(2) - 着色语言

### 1.数据类型
**标量：**bool,int,float
**向量：**vec2,vec3,vec4,ivec2,ivec3,ivec4,bvec2,bvec3,bvec4，数字代表该向量是几维向量，vec为float向量，ivec为int向量，bvec为bool向量。
将一个向量看为颜色时，最多使用4个分量：[argb]
将一个向量看为位置时，最多使用4个分量：[x,y,z,w]，w为齐次坐标，一般为1
将一个向量看为纹理时，最多使用4个分量：[s,t,p,q]
访问向量中分量的语法为:**<向量名>·<分量名>**，或者使用下标，如：
``` glsl
vec3 position = vec3(1,2,3);    //构造一个vec3的标量
position.x = 10;    //将x分量置为10
position[1] = 20;   //将y分量置为20
```
**矩阵：**mat2,mat3,mat4，都为float矩阵，数字代表几维矩阵
在OpenGL中，矩阵是按照列来组织的，所以可以将矩阵视为多个列向量的数组来访问，如：
一个矩阵赋值如下
```table
  第1列 | 第2列 | 第3列
1 | 2 | 3 
4 | 5 | 6
7 | 8 | 9
    
```
则在着色语言的访问方式：
```glsl
mat3 m3;
vec3 v1 = m3[0];    // v1为(1,4,7)
float f = m3[1][2];     // f为8
```
**采样器：**sampler2D,sampler3D,samplerCube
sampler2D：二维纹理采样器
sampler3D：三维纹理采样器
samplerCube：立方体贴图采样器
**结构体:**
```glsl
struct info {
    vec3 color;
    vec3 position;
    vec2 textureColor;
}
```
**数组**
**空类型**：void

### 2.初始化
```glsl
// 以下初始化方式都是正确的
float a = 2.3;
vec3 v1 = vec3(1,2,3);
vec3 v2 = vec3(v1.x, v1.y, 100);
vec4 v3 = vec4(v2, 200);
mat3 m1 = mat3(1, 2, 3, 4, 5, 6, 7, 8, 9);
mat3 m2 = mat3(v3, 4, 5, 6, 7, 8, 9);
mat3 m3 = mat3(1.0);
```
### 3.运算符之混合操作

```glsl
//以下混合操作方式都为正确的
vec4 color = vec4(0.7, 0.1, 0.5, 1.0);
vec3 temp = color.agb;
vec4 tempL = color.aabb;
tempLL.grb = color.aab;

//以下为不正确的
temp = color.xya;   //xy 和  a不在同一组，xy为位置分量，a为颜色分量
```
### 4.片元着色器中的浮点变量精度指定
顶点着色器直接声明变量就可以使用，但片元着色器中的变量需要手动指定，不然会编译错误，分为3种精度：
- lowp 低精度
- mediump 中精度
- highp 高精度
```glsl
lowp float color;
varying mediump vec2 position;
highp mat4 m;
```
如果希望片元着色器的float变量都使用同一种精度，则在最开始的地方声明：
```glsl
// precision <精度> <类型>
precision mediump float;
```

### 5.限定符
**attribute：**只能顶点着色器使用，由外部程序传入顶点着色器。
**uniform：**一致变量限定符，指的是在单个3D物体中所有顶点都相同的值，uniform可以用在顶点着色器和片元着色器中，它的值也是通过外部程序传入。
**varying：**易变变量，它修饰的变量可以从顶点着色器中传入片元着色器中，但是片元着色器真正获得的值会根据各个顶点传递的值，通过渲染管线的插值计算出每个片元的值。
**const：**常量限定，和C语言语法一致。

### 6.内建变量
**顶点着色器中的内建变量：**gl_Position,gl_PointSize等
- gl_Position(vec4)：顶点着色器从外部程序接收到顶点的原始坐标后，和变换矩阵计算得出的最终顶点坐标，需要赋给这个值，gl_Position赋值后会送入渲染管线后续流程继续处理，几乎在所有的顶点着色器中都需要给它赋值。
```glsl
uniform mat4 uMVPMatrix;    //变换矩阵
attribute vec3 aPosition;
void main() {
    gl_Position = uMVPMatrix * vec4(aPosition, 1);  //矩阵乘法，算出最终坐标
}
```
- gl_PointSize(float)：计算一个点的大小，默认为1

**片元着色器中的内建输入变量:**
- gl_FragCoord:内建输入变量，当前片元相对于窗口位置的坐标值x,y,z,1/w
![](./OpenGL-2-着色语言/test.png?r=40)

- gl_FrontFacing(bool)：判断当前正在处理的片元是否属于在光栅化阶段生成此片元的对应图元的正面，一般用于开发双面光照功能相关的应用程序中。
**片元着色器中的内建输出变量**
- gl_FragColor(vec4)：由片元着色器写入计算完成的片元的颜色值，可以通过 外部程序->顶点着色器->片元着色器传入。在OpenGL的纹理开发中，也可以通过外部程序传入纹理坐标后，通过纹理坐标去一张纹理图片上提取出颜色值。
- gl_FragData(vec4[])：变量值是一个数组。向gl_FragData[n]写入数据时为了指明后续固定功能管线要使用的片元数据n。如果后续固定功能管线要使用这个片元数据，但是着色器又没有给它写入值，那么片元数据将是一个未定义的值。

### 7.内建函数
着色器中的内建函数大致分为以下几种
- 角度转换和三角函数（radians、sin、cos...）
- 指数函数（pow、log...）
- 常见函数（abs、mod...）
- 几何函数（length、cross..）
- 矩阵函数（martrixCompMult..）
- 向量关系函数（lessThan、equal..）
- 纹理采样函数（texture2D、textureCube、texture3D...）
- 微分函数（dFdx..）