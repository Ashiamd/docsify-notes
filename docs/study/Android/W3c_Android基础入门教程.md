# Android基础入门教程

> https://www.w3cschool.cn/uawnhh/gi3sdozt.html

## 第一章-环境搭建和开发相关

### 1.2 开发环境搭建

1. 相关术语的解析

   >1. **Dalvik：** Android特有的虚拟机,和JVM不同,Dalvik虚拟机非常适合在移动终端上使用!
   >2. **AVD：** (android virtual machine):安卓虚拟设备,就是安卓的模拟器
   >3. **ADT：** (android development tools)安卓开发工具
   >4. **SDK：**(software development kit)软件开发工具包,就是安卓系统,平台架构等的工具集合,如adb.exe
   >5. **DDMS：**(dalvik debug monitor service)安卓调试工具
   >6. **adb：**安卓调试桥,在sdk的platform-tools目录下,功能很多,命令行必备
   >7. **DX工具：**将.class转换成.dex文件
   >8. **AAPT：**(android asset packing tool),安卓资源打包工具
   >9. **R.java文件：**由aapt工具根据App中的资源文件自动生成,可以理解为资源字典
   >10. **AndroidManifest.xml：**app包名 + 组件声明 + 程序兼容的最低版本 + 所需权限等程序的配置文件

#### 1.6 .9图片

> http://www.miued.com/2074/

### 1.8 工程相关解析(各种文件，资源访问)

1. res资源文件夹介绍：**res目录的资源文件会在R.javaw文件下生成对应的资源id，而assets目录需通过AssetManager以二进制流的形式读取**。

2. 图片加载时，如果你想禁止Android不跟随屏幕密度加载不同文件夹的资源,只需在AndroidManifest.xml文件中添加android:anyDensity="false"字段即可

3. values目录：

   > - demens.xml：定义尺寸资源
   > - string.xml：定义字符串资源
   > - styles.xml：定义样式资源
   > - colors.xml：定义颜色资源
   > - arrays.xml：定义数组资源
   > - attrs.xml：自定义控件时用的较多，自定义控件的属性！
   > - theme主题文件，和styles很相似，但是会对整个应用中的Actvitiy或指定Activity起作用，一般是改变窗口外观的！可在Java代码中通过setTheme使用，或者在Androidmanifest.xml中为添加theme的属性！ PS:你可能看到过这样的values目录：values-w820dp，values-v11等，前者w代表平板设备，820dp代表屏幕宽度；而v11这样代表在API(11)，即android 3.0后才会用到的！

4. raw目录： 

   用于存放各种原生资源(音频，视频，一些XML文件等)，我们可以通过openRawResource(int id)来获得资源的二进制流！其实和Assets差不多，不过这里面的资源会在R文件那里生成一个资源id而已

5. 通过R.java文件生成的资源id调用、访问资源：

   + Java代码中使用：

     > Java 文字：txtName.setText(getResources().getText(R.string.name)); 
     > 图片：imgIcon.setBackgroundDrawableResource(R.drawable.icon); 
     > 颜色：txtName.setTextColor(getResouces().getColor(R.color.red)); 
     > 布局：setContentView(R.layout.main);
     > 控件：txtName = (TextView)findViewById(R.id.txt_name);

   + XML代码中使用(通过@xxx即可获得)：

     > <TextView android:text="@string/hello_world" android:layout_width="wrap_content" android:layout_height="wrap_content" android:background = "@drawable/img_back"/>

### 1.9 Android程序签名打包

1. 签名类似于java的序列号，用于标记同一个应用程序。

   + 程序升级会检验证书签名是否一致；
   + 多个应用使用同一签证(模块化)，则系统把其识别为一个整体。

   + 代码ors数据共享，可以通过把它们在同一个证书签证范围内或者运行在同一进程来实现。感觉类似cookie的作用域domain和path效果

2. 到签名的apk文件目录下执行以下指令可确认应用是否已经签名

   ```shell
   jarsigner -verbose -certs -verify app-release.apk
   ```

### 1.10 反编译APK获取代码&资源

> https://www.jianshu.com/p/b5e3b1f3a42f 反编译apk查看源码

## 第二章-Android中的UI组件的详解

### 2.1 View和ViewGroup的概念

1. Android的所有用户界面都由View和ViewGroup构成。View绘制与用户交互的对象，ViewGroup是存放其他View和ViewGroup对象的布局容器。

#### 2.2.1 LinearLayout(线性布局)

1.Linear图解

![img](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5e739a62.jpg)

2. wrap_content和fill_parent(match_parent)的区别

   > Android布局基础知识：wrap_content,match_parent,layout_weight
   >
   > https://blog.csdn.net/m0_37828249/article/details/79066461
   >
   > Android 对Layout_weight属性完全解析以及使用ListView来实现表格
   >
   > https://blog.csdn.net/xiaanming/article/details/13630837
   >
   > Android layout_weight 计算方式详解
   >
   > https://blog.csdn.net/u012524598/article/details/22344765

   前者根据内容、高度设置；后者充满父组件的宽度or高度。

   以下是自己测试的一些情况：

   + 如果多个LinearLayout并列，假设3个，第一个wrap_content，那么后面出现match_parent，前面这个wrap_content就显示不了了；

     测试后发现，只要同级LinearLayout里出现match_parent，不管其他的wrap_content还是直接数值XXdp都是没法显示

   + 如果第一个wrap_content，后两个match_parent(都不设置权重)，那么其实第二个刚好占据屏幕，第三个跑到屏幕外

   即只要多个同级LinearLayout中只要出现一个match_parent，那么wrap_content就被盖住了，或者直接得不到空间。

   + **计算公式：自身width/height+（自身weight/所有控件weight和）*额外width/height**；
   + match_parent的本身width/height都是整个父组件的width/height；
   + 额外width/height即本身width/height - 所有组件的width/height
   + 要均等分可以0dp,然后设置各自的权重相同

   很多情况不用特地去记，不清楚动手试试就好了

3. 掌握xml配置属性和java对象设置属性的方法

   > java中设置weight属性

   ```java
   setLayoutParams(new LayoutParams(LayoutParams.FILL_PARENT,     
           LayoutParams.WRAP_CONTENT, 1)); 
   ```

4. 为LinearLayout设置分割线

   + 两种方式：(1)布局添加View标签，高度1px，则肉眼看为分割线;

     ​				   (2)使用LinearLayout的divider属性，需要指定分割线图片;

5. 一些注意点：
   + LinearLayout的gravity属性，设置该布局的对齐方式(左对齐、居中等等)
   +  **当 android:orientation="vertical" 时， 只有水平方向的设置才起作用，垂直方向的设置不起作用。 即：left，right，center_horizontal 是生效的。 当 android:orientation="horizontal" 时， 只有垂直方向的设置才起作用，水平方向的设置不起作用。 即：top，bottom，center_vertical 是生效的。** 

#### 2.2.2 RelativeLayout(相对布局)

1. 为什么会需要用到RelativeLayout(现在建议用constraintLayout代替，因为后者可以说是前者的升级版。复杂情况下ConstraintLayout可以减少布局嵌套深度，简单界面ConstraintLayout和其他布局差不多)

   + LinearLayout多层嵌套时明显会降低UI Render的效率，尤其是listView或者GridView上的item，效率会更低。过多层级的LinearLayout嵌套会占用更多的系统资源，可能引发stackoverflow；

     而RelativeLayout仅仅需要一层以父组件or兄弟组件参考+margin+padding就可以设置组件的显示位置。

     建议尽量搭配使用LinearLayout和RelativeLayout

2. 核心属性图

    ![2015-12-01_565da5eb6677d.jpg (1038×1500)](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5eb6677d.jpg) 

3. 尝试使用RelativeLayout

   + 绘制一个排列为十字形的5张图集合。
   
     + 我的代码：
   
       ```xml
       <RelativeLayout
           android:layout_height="match_parent"
           android:layout_width="match_parent"
           xmlns:android="http://schemas.android.com/apk/res/android">
           <ImageView
               android:id="@+id/test1"
               android:layout_width="100dp"
               android:layout_height="100dp"
               android:layout_centerInParent="true"
               android:src="@mipmap/test1"/>
           <ImageView
               android:layout_width="100dp"
               android:layout_height="100dp"
               android:layout_toStartOf="@+id/test1"
               android:src="@mipmap/test2"/>
           <ImageView
               android:layout_width="100dp"
               android:layout_height="100dp"
               android:layout_toEndOf="@id/test1"
               android:src="@mipmap/test3" />
           <ImageView
               android:layout_width="100dp"
               android:layout_height="100dp"
               android:layout_above="@id/test1"
               android:src="@mipmap/test4"/>
           <ImageView
               android:layout_width="100dp"
               android:layout_height="100dp"
               android:layout_below="@id/test1"
               android:src="@mipmap/test5"/>
       </RelativeLayout>
       ```
   
     + 我的效果图
   
       ![1571043727332](C:\Users\70408\AppData\Roaming\Typora\typora-user-images\1571043727332.png)
   
       + 教程代码：
   
         ```xml
         <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"    
             xmlns:tools="http://schemas.android.com/tools"    
             android:id="@+id/RelativeLayout1"    
             android:layout_width="match_parent"    
             android:layout_height="match_parent" >    
         
             <!-- 这个是在容器中央的 -->    
         
             <ImageView    
                 android:id="@+id/img1"     
                 android:layout_width="80dp"    
                 android:layout_height="80dp"    
                 android:layout_centerInParent="true"    
                 android:src="@drawable/pic1"/>    
         
             <!-- 在中间图片的左边 -->    
             <ImageView    
                 android:id="@+id/img2"     
                 android:layout_width="80dp"    
                 android:layout_height="80dp"    
                 android:layout_toLeftOf="@id/img1"    
                 android:layout_centerVertical="true"    
                 android:src="@drawable/pic2"/>    
         
             <!-- 在中间图片的右边 -->    
             <ImageView    
                 android:id="@+id/img3"     
                 android:layout_width="80dp"    
                 android:layout_height="80dp"    
                 android:layout_toRightOf="@id/img1"    
                 android:layout_centerVertical="true"    
                 android:src="@drawable/pic3"/>    
         
             <!-- 在中间图片的上面-->    
             <ImageView    
                 android:id="@+id/img4"     
                 android:layout_width="80dp"    
                 android:layout_height="80dp"    
                 android:layout_above="@id/img1"    
                 android:layout_centerHorizontal="true"    
                 android:src="@drawable/pic4"/>    
         
             <!-- 在中间图片的下面 -->    
             <ImageView    
                 android:id="@+id/img5"     
                 android:layout_width="80dp"    
                 android:layout_height="80dp"    
                 android:layout_below="@id/img1"    
                 android:layout_centerHorizontal="true"    
                 android:src="@drawable/pic5"/>    
         
         </RelativeLayout>
         ```
   
       + 教程效果图
   
          ![img](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5ee658f1.jpg)
   
   + margin和padding区别：和web一样的，margin就是元素外之间，padding是元素内之间
   
   + magrin的负数值运用：模拟进入软件后，弹出广告，**右上角的cancel按钮的margin是使用负数**的。其实就是右上角X符号位置的设置

#### 2.2.3 TableLayout(表格布局)

1. 学习路线

    ![2015-12-01_565da5f116a49.jpg (1044×299)](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5f116a49.jpg) 

2. 如何确定行数与列数

>①如果我们直接往TableLayout中添加组件的话,那么这个组件将占满一行！！！
>
>②如果我们想一行上有多个组件的话,就要添加一个TableRow的容器,把组件都丢到里面！
>
>③tablerow中的组件个数就决定了该行有多少列,而列的宽度由该列中最宽的单元格决定
>
>④tablerow的layout_width属性,默认是fill_parent的,我们自己设置成其他的值也不会生效！！！ 但是layout_height默认是wrap_content的,我们却可以自己设置大小！
>
>⑤整个表格布局的宽度取决于父容器的宽度(占满父容器本身)
>
>⑥有多少行就要自己数啦,一个tablerow一行,一个单独的组件也一行！多少列则是看tableRow中 的组件个数,组件最多的就是TableLayout的列数

3. 三个常用属性

> **android:collapseColumns:**设置需要**被隐藏**的列的序号
> **android:shrinkColumns:**设置允许**被收缩**的列的列序号
> **android:stretchColumns:**设置运行**被拉伸**的列的列序号
>
> 以上这三个属性的列号都是**从0开始算**的,比如shrinkColunmns = "2",对应的是第三列！
> 可以**设置多个**,用**逗号隔开**比如"0,2",如果是所有列**都生效**,则**用"*"号**即可
> 除了这三个常用属性,还有两个属性,分别就是跳格子以及合并单元格,这和HTML中的Table类似:
>
> **android:layout_column**="2":表示的就是**跳过**第二个,直接显示到第三个格子处,从1开始算的!
> **android:layout_span**="4":表示**合并**4个单元格,也就说这个组件占4个单元格

4. 属性使用介绍：
   + collapseColumns(隐藏列)，下标从0开始
   + stretchColumn(拉伸列)，下标从0开始，拉伸列填充剩余位置；多个会平分位置
   + shrinkColumn(收缩列)，下标从0开始，优先让别的拥有位置
   + android:layout_column="n"，跳过前n个位置,假如当前位置3，设置column=3，那么这个原本第三的位置会空出来，前2个元素位置不变，第三个及往后的元素往后往后移动一格
   + android:layout_span="n",该元素占据n格，其后的元素往后移动

5. 超级简易登录的Android界面

   + 我自己的代码：

     ```xml
     <TableLayout xmlns:android="http://schemas.android.com/apk/res/android"
         android:layout_width="match_parent"
         android:layout_height="match_parent">
         <TableLayout
             android:layout_height="match_parent"
             android:layout_width="match_parent"
             android:gravity="bottom"
             android:background="#9AFF9A">
             <TableRow
                 android:gravity="center">
                 <TextView android:text="@string/username"/>
                 <EditText/>
             </TableRow>
             <TableRow
                 android:gravity="center">
                 <TextView android:text="@string/password"/>
                 <EditText/>
             </TableRow>
             <TableRow
                 android:gravity="center">
                 <Button android:text="@string/login"/>
                 <Button android:text="@string/exit"/>
             </TableRow>
         </TableLayout>
     </TableLayout>
     ```

   + 效果图

     ![1571068499055](C:\Users\70408\AppData\Roaming\Typora\typora-user-images\1571068499055.png)

   + 教程的代码

     ```xml
     <TableLayout xmlns:android="http://schemas.android.com/apk/res/android"    
         xmlns:tools="http://schemas.android.com/tools"    
         android:id="@+id/TableLayout1"    
         android:layout_width="match_parent"    
         android:layout_height="match_parent"    
         tools:context=".MainActivity"     
         android:stretchColumns="0,3"    
         android:gravity="center_vertical"    
         android:background="#66FF66"    
         >    
     
         <TableRow>    
             <TextView />    
             <TextView     
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:text="用户名:"/>    
             <EditText     
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:minWidth="150dp"/>    
             <TextView />    
         </TableRow>    
     
         <TableRow>    
             <TextView />    
             <TextView     
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:text="密  码:"        
             />    
             <EditText     
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:minWidth="150dp"        
             />    
             <TextView />    
         </TableRow>    
     
         <TableRow>    
             <TextView />    
             <Button     
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:text="登陆"/>    
             <Button    
                 android:layout_width="wrap_content"    
                 android:layout_height="wrap_content"    
                 android:text="退出"/>    
             <TextView />    
         </TableRow>    
     
     </TableLayout>
     ```

   + 教程的效果图

      ![img](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5f20ba92.jpg) 

#### 2.2.4 FrameLayout(帧布局)

1. 属性：
   - **android:foreground:***设置改帧布局容器的前景图像
   - **android:foregroundGravity:**设置前景图像显示的位置

2. 随手指移动的图片

   + 看了参考代码，好奇手动bitmap.recycle()是不是类似file.close关闭物理资源;

     >是否需要主动调用Bitmap的recycle方法
     >
     > https://www.cnblogs.com/zhucai/p/5413340.html 

   + BItMap使用讲解

     > Android Bitmap详解
     >
     >  https://www.cnblogs.com/winter-is-coming/p/9112192.html 

   + Android中使用Canvas和Paint实现自定义View

     >Android中使用Canvas和Paint实现自定义View
     >
     > https://blog.csdn.net/Xiongjiayo/article/details/81456313 

   + Android动态添加View之addView的用法

     > Android动态添加View之addView的用法
     >
     >  https://www.jianshu.com/p/760573e1964f 

3. 跑动的人物图片：

   + Android基础夯实--你了解Handler有多少？

     >Android基础夯实--你了解Handler有多少？
     >
     > https://www.cnblogs.com/ryanleee/p/8204450.html 

4. 自己补充一点知识点：
   
   + Timer计时器是另外新建的一个独立子线程

#### 2.2.5 GridLayout(网格布局)

1. 相关属性总结图

    ![img](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5f58c9c8.jpg) 

2. 计算器简单布局的实现
   + 注意要点：
     +  代码很简单,只是回退与清除按钮横跨两列,而其他的都是直接添加的,默认每个组件都是 占一行一列,另外还有一点要注意的: 我们通过:**android:layout_rowSpan**与**android:layout_columnSpan**设置了组件横跨 多行或者多列的话,如果你要让组件填满横越过的行或列的话,需要添加下面这个属性: **android:layout_gravity = "fill"**！！！就像这个计算机显示数字的部分 
     + layout_gravity不要写成gravity，后者是该组件内的gravity，前者才是该组件在父组件中的gravity
     + 如果按顺序接下去第m行第n列，那么不需要特定指定元素是第几行or第几列

3. 用法总结

   ①GridLayout使用虚细线将布局划分为行,列和单元格,同时也支持在行,列上进行交错排列 

   ②使用流程: 

   > - step 1:先定义组件的对其方式 android:orientation 水平或者竖直,设置多少行与多少列
   > - step 2:设置组件所在的行或者列,记得是从0开始算的,不设置默认每个组件占一行一列
   > - step 3:设置组件横跨几行或者几列;设置完毕后,需要在设置一个填充:android:layout_gravity = "fill"

4. 使用GridLayout需要注意SDK版本问题，是Android4.0后才有的。低版本sdk使用GridLayout只需要导入v7包的GridLayout

   使用时标签< android.support.v7.widget.GridLayout >

#### 2.2.6 AbsoluteLayout(绝对布局)

1. 这个基本不会用到，毕竟绝对布局不适合多机型适配

   四大控制属性：

   >**①控制大小:** **android:layout_width:**组件宽度 **android:layout_height:**组件高度 
   >
   >**②控制位置:** **android:layout_x:**设置组件的X坐标 **android:layout_y:**设置组件的Y坐标 

2. <u>建议布局还是尽量使用LinearLayout的weight权重属性+RelativeLayout</u>

### 2.3 表单

#### 2.3.1 TextView(文本框)详解

+ 几个常用的单位介绍：

  > **dp(dip):** device independent pixels(设备独立像素). 不同设备有不同的显示效果,这个和设备硬件有关，一般我们为了支持WVGA、HVGA和QVGA 推荐使用这个，不依赖像素。
  >
  >  **px: pixels(像素)**. 不同设备显示效果相同，一般我们HVGA代表320x480像素，这个用的比较多。
  >
  >  **pt: point**，是一个标准的长度单位，1pt＝1/72英寸，用于印刷业，非常简单易用；
  >
  >  **sp: scaled pixels(放大像素)**. 主要用于字体显示best for textsize。

1. 几个常见属性：

   >- **id：**为TextView设置一个组件id，根据id，我们可以在Java代码中通过findViewById()的方法获取到该对象，然后进行相关属性的设置，又或者使用RelativeLayout时，参考组件用的也是id！
   >- **layout_width：**组件的宽度，一般写：**wrap_content**或者**match_parent(fill_parent)**，前者是控件显示的内容多大，控件就多大，而后者会填满该控件所在的父容器；当然也可以设置成特定的大小，比如我这里为了显示效果，设置成了200dp。
   >- **layout_height：**组件的宽度，内容同上。
   >- **gravity：**设置控件中内容的对齐方向，TextView中是文字，ImageView中是图片等等。
   >- **text：**设置显示的文本内容，一般我们是把字符串写到string.xml文件中，然后通过@String/xxx取得对应的字符串内容的，这里为了方便我直接就写到""里，不建议这样写！！！
   >- **textColor：**设置字体颜色，同上，通过colors.xml资源来引用，别直接这样写！
   >- **textStyle：**设置字体风格，三个可选值：**normal**(无效果)，**bold**(加粗)，**italic**(斜体)
   >- **textSize：**字体大小，单位一般是用sp！
   >- **background：**控件的背景颜色，可以理解为填充整个控件的颜色，可以是图片哦！

   + TextView常用属性(网文)

     >  https://www.cnblogs.com/xqz0618/p/textview.html 

2. 实际开发例子

   2.1 带阴影的TextView

   - **android:shadowColor:**设置阴影颜色,需要与shadowRadius一起使用哦!
   - **android:shadowRadius:**设置阴影的模糊程度,设为0.1就变成字体颜色了,建议使用3.0
   - **android:shadowDx:**设置阴影在水平方向的偏移,就是水平方向阴影开始的横坐标位置
   - **android:shadowDy:**设置阴影在竖直方向的偏移,就是竖直方向阴影开始的纵坐标位置

   2.2 带边框的TextView

   + TextView其实是很多控件的父类(比如Button)，实现边框的原理就是编写一个ShapeDrawable的资源文件，然后TextView将background设置为该资源文件。

+ 突然好奇Drawable是不是都存放xml为主，然后mipmap存放png等图片，找到了一篇对drawable和mipmap目录的使用的讲解文章：

  >  https://www.jianshu.com/p/f7dc272b3469 

  	+ 如果需要根据不同机型自动选择合适的图片来缩小显示以获得较清晰的launcher图标视图，就应该把图片存放在mipmap中。

   + 标准的launcher中，默认访问当前设备destiny对应的drawable目录获取资源，并不会判断是否需要获取更高密度的drawable，而定制的launcher才会知道自己图标比标准的launcher大，所以开发者才会尝试获取更高密度的资源来使用；

     **而放在mipmap中，标准的launcher会自动去获取更加合适的icon**

   + 同样条件下，Activity内Drawable和mipmap加载图片都是直接选取正好合适的密度的文件夹资源；而Launcher内的mipmap则会获取更高destiny的图片缩小处理以获取最清晰的图标。

     > **结论：在App中，无论你将图片放在drawable还是mipmap目录，系统只会加载对应density中的图片，例如xxhdpi的设备，只会加载drawable-xxhdpi或者mipmap-xxhdpi中的资源。
     > 而在Launcher中，如果使用mipmap，那么Launcher会自动加载更加合适的密度的资源。**
     >
     > **或者说，mipmap会自动选择更加合适的图片仅在launcher中有效。**

  + <u>把图片放在mipmap中并不能保证放大、缩小的操作下图片质量会比drawable好。</u>

  + drawable和mipmap目录的结论：

    >1. **App中，无论你将图片放在drawable还是mipmap目录，系统只会加载对应density中的图片。
    >   而在Launcher中，如果使用mipmap，那么Launcher会自动加载更加合适的密度的资源。**
    >2. **应用内使用到的图片资源，并不会因为你放在mipmap或者drawable目录而产生差异。单纯只是资源路径的差异R.drawable.xxx或者R.mipmap.xxx。（也可能在低版本系统中有差异）**
    >3. **一句话来说就是，自动跨设备密度展示的能力是launcher的，而不是mipmap的。**
    >
    >总的来说，app图标（launcher icon) 必须放在mipmap目录中，并且最好准备不同密度的图片，否则缩放后可能导致失真。
    >
    >而应用内使用到的图片资源，放在drawable目录亦或是mipmap目录中是没有区别的，该准备多个密度的还是要准备多个密度，如果只想使用一份切图，那尽量将切图放在高密度的文件夹中。

  + Android图片加载时引用一些规则，下面简单描述(摘自上面提到的网文)

    > 当我们使用资源id来去引用一张图片时，Android会使用一些规则来去帮我们匹配最适合的图片。什么叫最适合的图片？比如我的手机屏幕密度是xxhdpi，那么drawable-xxhdpi文件夹下的图片就是最适合的图片。因此，当我引用android_logo这张图时，如果drawable-xxhdpi文件夹下有这张图就会优先被使用，在这种情况下，图片是不会被缩放的。但是，如果drawable-xxhdpi文件夹下没有这张图时， 系统就会自动去其它文件夹下找这张图了，优先会去更高密度的文件夹下找这张图片，我们当前的场景就是drawable-xxxhdpi文件夹，然后发现这里也没有android_logo这张图，接下来会尝试再找更高密度的文件夹，发现没有更高密度的了，这个时候会去drawable-nodpi文件夹找这张图，发现也没有，那么就会去更低密度的文件夹下面找，依次是drawable-xhdpi -> drawable-hdpi -> drawable-mdpi -> drawable-ldpi。 

+ Launcher简介

  >  https://www.ituring.com.cn/book/tupubarticle/8498 

+ Android中的Drawable基础与自定义Drawable

  >  https://blog.csdn.net/feather_wch/article/details/79124608 

  + Drawable抽象类，可以通过XML定义or代码创建，**颜色**、图片等都可以是Drawable

  + Drawable比自定义View成本低，非图片类的Drawable所占空间小

  + 内部宽高(即图片的宽高)不等于Drawable的大小(其无大小概念，毕竟抽象类。作为View背景时，会被拉伸至View的大小)，颜色Drawable没有内部宽高。

  + NinePatchDrawable(.9图片)的作用

    ​	自动根据宽高缩放不失真；实际使用，可以直接引用图片或者通过XML描述

+ Android app 性能优化之视图优化

  >  https://blog.csdn.net/shenjinalin123/article/details/51982794 

  + VSync是什么，什么时候用到

    >  https://www.sohu.com/a/325643652_120099902 

    + 一个图像的两个不同“显示器”互相碰撞时，因为游戏FPS传递的信息是显示器的刷新率跟不上的，会使得视图错位、碎片化。这种情况经常发生在游戏帧速要求高(>=60Hz)但显示器的刷新率不真正超过60Hz的场景。如果一个显示器和游戏在同步上有问题，那么V-Sync可能会将FPS进一步降低到30FPSS...

  + 深入理解线性空间与HDR

    >  https://www.jianshu.com/p/631a14877f4e 

  + 深入了解HDR,让你认识真正的HDR照片

    >  https://baijiahao.baidu.com/s?id=1606763887374415267&wfr=spider&for=pc 

    + 基本上HDR就是明暗处理，使得图像尽可能接近人眼所观察的明暗效果。

+ ShapeDrawable

  >  https://www.jianshu.com/p/0d8a09634f7a 

  

  2.3 带图片(drawableXxx)的TextView

  + **利用drawableTop/Buttom/Left/Right属性设置图片位置**(用LinearLayout的话嵌套越多，性能越差)，但是这种方式无法直接在XML上修改图片大小，需要在java代码中设置大小	

  2.4 使用autoLink属性识别连接属性

  >  当文字中出现了URL，E-Mail，电话号码，地图的时候，我们可以通过设置autoLink属性；当我们点击 文字中对应部分的文字，即可跳转至某默认APP，比如一串号码，点击后跳转至拨号界面！ 

   ![2015-12-01_565da5fc3f9c0.jpg (376×229)](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da5fc3f9c0.jpg) 


#### 2.3.2 EditText(输入框)详解

1. GridLayout某行单元格使用layout_gravity=“fill”使得该列充满这个单元格

2. 提示文本的属性：

   ```xml
   android:hint="默认提示文本" <!-- (设置提示的文本内容) -->
   android:textColorHint="#95A1AA" <!-- (设置提示文本的颜色) -->
   ```

3. 设置获取焦点后全选组件内所有文本内容

   ```xml
   android:selectAllOnFocus="true"
   ```

4. 限制EditText输入类型(应该是预设了正则表达式判断)

   通过android:inputType="XXX"来限制输入类型

   比如设置android:inputType="phone",点击EditText就会直接显示数字键盘

   ![1571549236911](C:\Users\70408\AppData\Roaming\Typora\typora-user-images\1571549236911.png)

   + android:inputType的可选参数：

     + 文本类型(多为大写、小写和数字符号):

     ```xml
     android:inputType="none"  
     android:inputType="text"  
     android:inputType="textCapCharacters"  
     android:inputType="textCapWords"  
     android:inputType="textCapSentences"  
     android:inputType="textAutoCorrect"  
     android:inputType="textAutoComplete"  
     android:inputType="textMultiLine"  
     android:inputType="textImeMultiLine"  
     android:inputType="textNoSuggestions"  
     android:inputType="textUri"  
     android:inputType="textEmailAddress"  
     android:inputType="textEmailSubject"  
     android:inputType="textShortMessage"  
     android:inputType="textLongMessage"  
     android:inputType="textPersonName"  
     android:inputType="textPostalAddress"  
     android:inputType="textPassword"  
     android:inputType="textVisiblePassword"  
     android:inputType="textWebEditText"  
     android:inputType="textFilter"  
     android:inputType="textPhonetic" 
     ```

     + 数值类型：

     ```xml
     android:inputType="number"  
     android:inputType="numberSigned"  
     android:inputType="numberDecimal"  
     android:inputType="phone"//拨号键盘  
     android:inputType="datetime"  
     android:inputType="date"//日期键盘  
     android:inputType="time"//时间键盘
     ```

5. 设置最小行，最多行，单行，多行，自动换行

   EditText默认是多行显示，自动换行（宽度不够时会自动换行显示）

   可以自定义最小行数、最大行数；

   ```xml
   android:minLines="m" <!-- 就是没有内容也会预留3行，输入前在第1行，下面有2行空白 -->
   android:maxLines="n" <!-- 超过时，文字会自动向上滚动(即最多显示3行，有隐藏的scroller) -->
   ```

6. 设置文字单行显示，不换行

   ```xml
   android:singleLine="true"
   ```

7. 设置字与字之间的间隔

   ```xml
   android:textScaleX="1.5"<!-- 水平间隔 -->
   android:textScaleY="1.5"<!-- 垂直间隔 -->
   ```

8. 英文字母大写类型属性设置: android:capitalize 默认none

   ```xml
   sentences：仅第一个字母大写
   words：每一个单词首字母大小，用空格区分单词
   characters:每一个英文字母都大写
   ```

9. 控制EditText四周的间隔距离与内部文字与边框间的距离

   >  我们使用**margin**相关属性增加组件相对其他控件的距离，比如android:marginTop = "5dp" 使用**padding**增加组件内文字和组件边框的距离，比如android:paddingTop = "5dp" 

10. 设置EditText获得焦点，同时弹出小键盘

    需求背景：进入Activity后让EditText获得焦点，同时弹出小键盘供用户输入。

    + EditText获得焦点和清除焦点

      > edit.**requestFocus**(); //请求获取焦点
      > edit.**clearFocus**(); //清除焦点 

    + 低版本的系统调用requestFocus()就自动弹出小键盘了，稍微高的版本需要代码调用小键盘

      可以通过在manifests/AndroidManifest.xml中为<activiy》设置android:windowSoftInputMode属性来解决这个问题。

      >**android:windowSoftInputMode** Activity主窗口与软键盘的交互模式，可以用来避免输入法面板遮挡问题，Android1.5后的一个新特性。
      >这个属性能影响两件事情：
      >【一】当有焦点产生时，软键盘是隐藏还是显示
      >【二】是否减少活动主窗口大小以便腾出空间放软键盘
      >
      >简单点就是有焦点时的键盘控制以及是否减少Act的窗口大小，用来放小键盘
      >有下述值可供选择，可设置多个值，用"|"分开
      >**stateUnspecified**：软键盘的状态并没有指定，系统将选择一个合适的状态或依赖于主题的设置
      >**stateUnchanged**：当这个activity出现时，软键盘将一直保持在上一个activity里的状态，无论是隐藏还是显示
      >**stateHidden**：用户选择activity时，软键盘总是被隐藏
      >**stateAlwaysHidden**：当该Activity主窗口获取焦点时，软键盘也总是被隐藏的
      >**stateVisible**：软键盘通常是可见的
      >**stateAlwaysVisible**：用户选择activity时，软键盘总是显示的状态
      >**adjustUnspecified**：默认设置，通常由系统自行决定是隐藏还是显示
      >**adjustResize**：该Activity总是调整屏幕的大小以便留出软键盘的空间
      >**adjustPan**：当前窗口的内容将自动移动以便当前焦点从不被键盘覆盖和用户能总是看到输入内容的部分

      我们可以在AndroidManifest.xml为需要弹出小键盘的Activity设置这个属性，比如：

      ![img](https://atts.w3cschool.cn/attachments/image/cimg/2015-12-01_565da604a8b67.jpg)

      然后在EditText对象requestFocus()就可以了~

      ps:我自己试了一下网上的键盘工具类，也是莫名其妙不起作用，文章这个就可以

11. EditText光标位置的控制