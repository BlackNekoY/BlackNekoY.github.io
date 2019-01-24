---
title: OpenGL(3) - OpenGL's Hello World
tags: 
  - OpenGL
comments: true
date: 2017-09-15 17:50:36
categories:
  - 技术
description: OpenGL学习笔记 3
---

# OpenGL(3) - OpenGL's Hello World
### 1.编写GLSL程序
着色器是通过glsl语言编写的，各个平台统一，实现了一次编写，到处使用。在Android中，glsl程序一般有两种书写方式：
- 单独写成文件，放在/app/assets/ 文件夹下，后缀名为glsl，在代码中通过Resource读取内容。
```glsl
// vertex.glsl

uniform mat4 uMVPMatrix;
attribute vec3 aPosition;
attribute vec4 aColor;
varying vec4 vColor;
void main() {
    gl_Position = uMVPMatrix * vec4(aPosition, 1);
    vColor = aColor;
}
```
- 在代码中通过字符串拼接生成final的shader代码
```java
  private final String VERTEX_SHADER_CODE =
            "uniform mat4 uMVPMatrix;\n" +
            "attribute vec3 aPosition;\n" +
            "attribute vec2 aTexturePos;\n" +  //传给顶点着色器的纹理坐标
            "varying vec2 vTexturePos;\n" +     //传给片元着色器的纹理坐标
            "void main() {\n" +
            "    gl_Position = uMVPMatrix * vec4(aPosition, 1);\n" +
            "    vTexturePos = aTexturePos;\n" +
            "}";
```
### 2.创建着色器程序的工具类
通用方法，写成工具类。一般在GLSurfaceView的onCreate中或者在可以被draw的Model类中调用。
```java
// ShaderUtil.java
 /**
     * 创建着色器程序
     * @param vertexSource 顶点着色器源代码
     * @param fragmentSource 片元着色器源代码
     * @return 程序id
     */
    public static int createProgram(String vertexSource, String fragmentSource) {
        //加载顶点着色器和片元着色器
        int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertexSource);
        if(vertexShader == 0) {
            return 0;
        }

        int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentSource);
        if(fragmentShader == 0) {
            return 0;
        }

        //创建着色器程序并attach着色器
        int program = GLES20.glCreateProgram(); 
        if(program != 0) {
            GLES20.glAttachShader(program, vertexShader);
            checkGlError("glAttachShader");
            GLES20.glAttachShader(program, fragmentShader);
            checkGlError("glAttachShader");
            GLES20.glLinkProgram(program);
            int[] linkStatus = new int[1];
            GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, linkStatus, 0);
            if(linkStatus[0] != GLES20.GL_TRUE) {
                Log.e(TAG, "link program error:" + GLES20.glGetProgramInfoLog(program));
                program = 0;
            }
        }
        return program;
    }
    
 /**
     * 加载着色器
     * @param shaderType 着色器类型
     *                   <p>GLES20.GL_VERTEX_SHADER:顶点着色器
     *                   <p>GLES20.GL_FRAGMENT_SHADER:片元着色器<p>
     * @param source 着色器的glsl代码
     * @return 着色器id
     */
    public static int loadShader(int shaderType, String source) {
        int shader = GLES20.glCreateShader(shaderType); //根据着色器type创建一个着色器，返回其id
        if(shader != 0) {
            GLES20.glShaderSource(shader, source); //将着色器与源代码绑定
            GLES20.glCompileShader(shader);     //编译
            int[] compiled = new int[1];

            // 获取Shader的编译情况
            GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, compiled, 0);
            if(compiled[0] == 0) {
                Log.e(TAG, "compile error:" + shaderType);
                Log.e(TAG, "compile error:" + GLES20.glGetShaderInfoLog(shader));
                GLES20.glDeleteShader(shader);
            }
        }
        return shader;
    }
```

### 3.顶点着色器和片元着色器
```glsl
// vertex.glsl  顶点着色器
uniform mat4 uMVPMatrix;        //变换矩阵
attribute vec3 aPosition;              //初始顶点坐标
attribute vec4 aColor;                  //顶点颜色
varying vec4 vColor;                    //传递给片元着色器的顶点颜色
void main() {
    gl_Position = uMVPMatrix * vec4(aPosition, 1);      //将初始顶点和变换矩阵做矩阵乘法，得到变换后的顶点坐标，赋给内建变量
    vColor = aColor;                                    //vColor传递给片元着色器
}
```
```glsl
// fragment.glsl    片元着色器
precision mediump float;    //指定片元着色器float精度为中
varying vec4 vColor;        //由顶点传入的经过插值化的颜色值
void main() {
    gl_FragColor = vColor;
}
```

### 4.Model类开发
在外部程序中获取glsl程序中该变量对应id->初始化缓冲区->draw的时候将缓冲区数据送入渲染管线。
```java
//三角形的Model
public class Triangle {

    public static final String TAG = "Triangle";

    /**
     * 具体物体的3D变换矩阵
     */
    static float[] mMMartrix = new float[16];

    int mProgram;   //程序id
    /**
     * 顶点着色器中 uniform mat4 uMVPMatrix 变量id
     */
    int muMVPMatrixHandle;
    /**
     * 顶点着色器中 attribute vec3 aPosition 变量id
     */
    int maPositionHandle;
    /**
     * 顶点着色器中 attribute vec4 aColor 变量id
     */
    int maColorHandle;
    /**
     * 顶点着色器代码
     */
    String mVertexShader;
    /**
     * 片元着色器代码
     */
    String mFragmentShader;
    /**
     * 初始顶点数据的缓冲区
     */
    FloatBuffer mVertexBuffer;
    /**
     * 初始颜色数据的缓冲区
     */
    FloatBuffer mColorBuffer;
    /**
     * 顶点个数
     */
    int vCount = 0;
    /**
     * 当前绕X轴旋转的角度
     */
    float xAngle;

    public Triangle(Resources resources) {
        initVertexData();
        initShader(resources);
    }

    /**
     * Array 长度分配和顶点个数有关系：顶点个数 * 每个顶点需要多少个分量,vec3就是3个分量
     * ByteBuffer长度分配和Array长度有关系：Array长度 * Array一个元素占多少个字节(一个float为4个字节)
     */
    private void initVertexData() {
        vCount = 3;
        final float UNIT_SIZE = 0.2f;

        //顶点位置数据： 3个顶点，每个顶点位置为(x,y,z)的vec3数据
        float[] vertexArray = new float[]{
                -4*UNIT_SIZE, 0, 0, //顶点1
                0, -4*UNIT_SIZE, 0, //顶点2
                4*UNIT_SIZE,0 , 0   //顶点3
        };
        mVertexBuffer = ByteBuffer.allocateDirect(vertexArray.length * 4)
                .order(ByteOrder.nativeOrder()) //之所以不直接用FloatBuffer.allocate方法是因为只有ByteBuffer有nativeOrder()方法
                .asFloatBuffer()
                .put(vertexArray);
        mVertexBuffer.position(0);

        //顶点颜色数据： 3个顶点，每个顶点颜色为(r,g,b,a)的vec4数据
        float[] colorArray = new float[]{
                1, 0, 0, 0, //顶点1
                0, 0, 1, 0, //顶点2
                0, 1, 0, 0  //顶点3
        };
        mColorBuffer = ByteBuffer.allocateDirect(colorArray.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer()
                .put(colorArray);
        mColorBuffer.position(0);
    }

    private void initShader(Resources resources) {
        // 从glsl文件中加载着色器代码，当然也可以在Java代码中写死
        mVertexShader = ShaderUtil.loadFromAssetsFile("vertex.glsl", resources);
        mFragmentShader = ShaderUtil.loadFromAssetsFile("fragment.glsl", resources);

        if (TextUtils.isEmpty(mVertexShader) || TextUtils.isEmpty(mFragmentShader)) {
            Log.e(TAG, "load vertexShader or fragmentShader source code failed.");
            Log.e(TAG, "vertexShader: " + mVertexShader);
            Log.e(TAG, "fragmentShader: " + mFragmentShader);
            return;
        }

        mProgram = ShaderUtil.createProgram(mVertexShader, mFragmentShader);
        if(mProgram == 0) {
            Log.e(TAG, "createProgram failed.");
            return;
        }
        //获取需要java层传给GL层的变量的id
        maPositionHandle = GLES20.glGetAttribLocation(mProgram, "aPosition");
        maColorHandle = GLES20.glGetAttribLocation(mProgram, "aColor");
        muMVPMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");
    }
    // 使用OpenGL画出自己，一般在GLSurfaceView.Render的onDrawFrame中调用
    public void drawSelf() {
        GLES20.glUseProgram(mProgram);
       //在每次draw的时候初始化变换矩阵，然后进行矩阵变换，如果不调用会出现以下两种情况
       //①变换矩阵为全0矩阵，无法渲染
       //②变换矩阵还保留着上一次的值，渲染异常
        Matrix.setRotateM(mMMartrix, 0, 0, 0, 1, 0);    
         // 对物体顶点进行各种变换，当然也可以不操作直接把原始顶点画上去。
        Matrix.rotateM(mMMartrix, 0, xAngle, 1, 0, 0);

        //对于attribute变量，需要enable
        GLES20.glEnableVertexAttribArray(maPositionHandle);
        GLES20.glEnableVertexAttribArray(maColorHandle);

        //将需要传给着色器的变量通过id送入渲染管线
        GLES20.glUniformMatrix4fv(muMVPMatrixHandle, 1, false, MatrixState.getFinalMatrix(mMMartrix), 0);   //getFinalMatrix是将物体本来的变换矩阵与摄像机矩阵、投影矩阵相乘而得出的最终的变换矩阵，这个矩阵已经是投影到近平面上的矩阵了，关于摄像机和投影可以看下篇文章
        GLES20.glVertexAttribPointer(
        maPositionHandle,   //顶点着色器变量id
        3,                  //着色器中该变量有几个分量,aPosition是vec3，所以是3个
        GLES20.GL_FLOAT,    //着色器中该变量类型,vec是float型
        false,
        3*4,                //一次从Buffer中取多少数据作为该变量的值。这里一个vec3顶点有3个分量，每个分量为float型，所以每次取值跨越的字节数为 3*4bytes = 12bytes
        mVertexBuffer);     //缓冲区
        GLES20.glVertexAttribPointer(maColorHandle, 4, GLES20.GL_FLOAT, false, 4*4, mColorBuffer);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vCount);
    }
}
```
### 5.GLSurfaceView
要使用OpenGL渲染，需要GLSurfaceView的EGL环境，这里通过继承GLSurfaceView实现。
```java
public class MySurfaceView extends GLSurfaceView {

    final float ANGLE_SPAN = 0.375f;
    RotateThread rthread;
    SceneRender mRender;

    public MySurfaceView(Context context) {
        this(context, null);
    }

    public MySurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setEGLContextClientVersion(2);  //设置OpenGL版本为2.x
        mRender = new SceneRender();
        setRenderer(mRender);
        // 设置渲染模式，RENDERMODE_CONTINUOUSLY为底层不停调用onDrawFrame进行渲染
        //  RENDERMODE_WHEN_DIRTY为只有当onSurfaceCreated时和主动调用requestRender时才会发起渲染
        setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    }


    private class SceneRender implements GLSurfaceView.Renderer {

        Triangle tle;

        @Override
        public void onSurfaceCreated(GL10 gl, EGLConfig config) {
            GLES20.glClearColor(0, 0, 0, 1.0f); // 屏幕背景色
            GLES20.glEnable(GLES20.GL_DEPTH_TEST);  // 开启深度检测
            tle = new Triangle(getResources());
            rthread = new RotateThread();
            rthread.start();
        }

        @Override
        public void onSurfaceChanged(GL10 gl, int width, int height) {
            GLES20.glViewport(0, 0, width, height); //设置视口
            float ratio = (float)width / height;
            MatrixState.setProjectFrustum(-ratio, ratio, -1, 1, 2, 10); // 设置投影矩阵：透视投影
            MatrixState.setCamera(  //设置摄像机矩阵
                    0, 0, 3,
                    0f, 0f, 0f,
                    0f, -1f, 0f);
            //关于投影矩阵和摄像机矩阵，可以看文章：摄像机和投影
        }

        @Override
        public void onDrawFrame(GL10 gl) {
            // 每一次都需要清除存在于帧缓冲中的上一次的深度缓存和颜色缓存，不然会将上一帧的内容也画上去
            GLES20.glClear(GLES20.GL_DEPTH_BUFFER_BIT | GLES20.GL_COLOR_BUFFER_BIT);    
            tle.drawSelf();
        }
    }

    private class RotateThread extends Thread {
        public boolean flag = true;

        @Override
        public void run() {
            while (flag) {
                //通过不停改变xAngle达到一种绕x轴旋转的效果
                mRender.tle.xAngle += ANGLE_SPAN;
                try {
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```