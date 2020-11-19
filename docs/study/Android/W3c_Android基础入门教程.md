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
    >     而在Launcher中，如果使用mipmap，那么Launcher会自动加载更加合适的密度的资源。**
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

    >EditText提供setSelection()移动光标/选中部分文本
    >
    >setSelection(int index) 设置光标位置; 
    >
    >setSelection(int start,int stop)部分选中;
    >
    >使用setSelectAllOnFocus(true),让EditText获得焦点时选中所有内容

12. 带表情的EditText的简单实现

    使用SpannableString

13. 带删除按钮的EditText

    为EditText设置addTextChangedListener，然后重写TextWatcher（）里的抽象方法，这个用于监听输入框变化的；然后setCompoundDrawablesWithIntrinsicBounds设置小叉叉的图片；最后，重写onTouchEvent方法，如果点击区域是小叉叉图片的位置，清空文本！ 

#### 2.3.3 Button(按钮)与ImageButton(图像按钮)

> Button是TextView的子类。实际开发，按钮基本就是点击或弹起时变色。通过StateListDrawable这种Drawable资源来实现

1. StateListDrawable简介：

   > StateListDrawable是Drawable资源的一种，可以根据不同的状态，设置不同的图片效果，我们只需要讲Button的blackground属性设置为该drawable资源即可轻松实现，按下 按钮时不同的按钮颜色或背景！ 

   可设置属性

   >- **drawable**:引用的Drawable位图,我们可以把他放到最前面,就表示组件的正常状态~
   >- **state_focused**:是否获得焦点
   >- **state_window_focused**:是否获得窗口焦点
   >- **state_enabled**:控件是否可用
   >- **state_checkable**:控件可否被勾选,eg:checkbox
   >- **state_checked**:控件是否被勾选
   >- **state_selected**:控件是否被选择,针对有滚轮的情况
   >- **state_pressed**:控件是否被按下
   >- **state_active**:控件是否处于活动状态,eg:slidingTab
   >- **state_single**:控件包含多个子控件时,确定是否只显示一个子控件
   >- **state_first**:控件包含多个子控件时,确定第一个子控件是否处于显示状态
   >- **state_middle**:控件包含多个子控件时,确定中间一个子控件是否处于显示状态
   >- **state_last**:控件包含多个子控件时,确定最后一个子控件是否处于显示状态

#### 2.3.4 ImageView(图像视图)

主要需要了解的内容如下：

+ ImageView的src属性和blackground的区别；
+ adjustViewBounds设置图像缩放时是否按长宽比
+ scaleType设置缩放类型
+ 最简单的绘制圆形的ImageView

1. src属性和background属性的区别

   >在API文档中我们发现ImageView有两个可以设置图片的属性，分别是：src和background
   >
   >**常识：**
   >
   >①background通常指的都是**背景**,而src指的是**内容**!!
   >
   >②当使用**src**填入图片时,是按照图片大小**直接填充**,并**不会进行拉伸**
   >
   >而使用background填入图片,则是会根据ImageView给定的宽度来进行**拉伸**

   我自己试了以下，src所在空间小的时候图片按原本比例缩小，所在空间大时，不变化；

   background随着所在空间被拉伸(所以最好用.9.png图片，这样拉伸时不失真)

2. 解决background拉伸导致图片变形的方法
   + 如果动态加载，添加View时把大小写死
   + xml布局方式，在drawable目录下新建xml使用\<bitmap\>位图对象

3. 设置透明度

4. Java代码中设置blackground和src属性

   >  前景(对应src属性):**setImageDrawable**( );
   > 背景(对应background属性):**setBackgroundDrawable**( ); 

5. 使用adjustViewBounds设置缩放是否按照原图长宽比

   >ImageView为我们提供了**adjustViewBounds**属性，用于设置缩放时是否保持原图长宽比！ 单独设置不起作用，需要配合**maxWidth**和**maxHeight**属性一起使用！而后面这两个属性 也是需要adjustViewBounds为true才会生效的~
   >
   >- android:maxHeight:设置ImageView的最大高度
   >- android:maxWidth:设置ImageView的最大宽度

6. scaleType设置缩放类型

   >android:scaleType用于设置显示的图片如何缩放或者移动以适应ImageView的大小 Java代码中可以通过imageView.setScaleType(ImageView.ScaleType.CENTER);来设置~ 可选值如下：
   >
   >- **fitXY**:对图像的横向与纵向进行独立缩放,使得该图片完全适应ImageView,但是图片的横纵比可能会发生改变
   >- **fitStart**:保持纵横比缩放图片,知道较长的边与Image的编程相等,缩放完成后将图片放在ImageView的左上角
   >- **fitCenter**:同上,缩放后放于中间;
   >- **fitEnd**:同上,缩放后放于右下角;
   >- **center**:保持原图的大小，显示在ImageView的中心。当原图的size大于ImageView的size，超过部分裁剪处理。
   >- **centerCrop**:保持横纵比缩放图片,知道完全覆盖ImageView,可能会出现图片的显示不完全
   >- **centerInside**:保持横纵比缩放图片,直到ImageView能够完全地显示图片
   >- **matrix**:默认值，不改变原图的大小，从ImageView的左上角开始绘制原图， 原图超过ImageView的部分作裁剪处理

#### 2.3.5 RadioButton(单选按钮)&Checkbox(复选框)

> 本节给大家带来的是Andoird基本UI控件中的RadioButton和Checkbox; 先说下本节要讲解的内容是：RadioButton和Checkbox的
>**1.基本用法
>2.事件处理；
>3.自定义点击效果；
>4.改变文字与选择框的相对位置；
>5.修改文字与选择框的距离** 

---

##### 1.RadioButton

>  如题单选按钮，就是只能够选中一个，所以我们需要把RadioButton放到RadioGroup按钮组中，从而实现 单选功能！先熟悉下如何使用RadioButton，一个简单的性别选择的例子： 另外我们可以为外层RadioGroup设置orientation属性然后设置RadioButton的排列方式，是竖直还是水平~ 

##### 2.改变文字与选择框的相对位置

>这个实现起来也很简单，还记得我们之前学TextView的时候用到的drawableXxx吗？ 要控制选择框的位置，两部即可！设置：
>
>**Step 1.** android:button="@null"
>**Step 2.** android:drawableTop="@android:drawable/btn_radio"
>当然我们可以把drawableXxx替换成自己喜欢的效果！

##### 3. 修改文字与选择框的距离

>  有时，我们可能需要调节文字与选择框之间的距离，让他们看起来稍微没那么挤，我们可以：
> 1.在XML代码中控制： 使用android:paddingXxx = "xxx" 来控制距离
> 2.在Java代码中，稍微好一点，动态计算paddingLeft! 

```java
rb.setButtonDrawable(R.drawable.rad_btn_selctor);
int rb_paddingLeft = getResources().getDrawable(R.mipmap.ic_checkbox_checked).getIntrinsicWidth()+5; 
rb.setPadding(rb_paddingLeft, 0, 0, 0);
```

####  2.3.6 开关按钮ToggleButton和开关Switch

#### 2.3.7 ProgressBar

常用属性：

> - android:**max**：进度条的最大值
> - android:**progress**：进度条已完成进度值
> - android:**progressDrawable**：设置轨道对应的Drawable对象
> - android:**indeterminate**：如果设置成true，则进度条不精确显示进度
> - android:**indeterminateDrawable**：设置不显示进度的进度条的Drawable对象
> - android:**indeterminateDuration**：设置不精确显示进度的持续时间
> - android:**secondaryProgress**：二级进度条，类似于视频播放的一条是当前播放进度，一条是缓冲进度，前者通过progress属性进行设置！

对应的java方法：

> - **getMax**()：返回这个进度条的范围的上限
> - **getProgress**()：返回进度
> - **getSecondaryProgress**()：返回次要进度
> - **incrementProgressBy**(int diff)：指定增加的进度
> - **isIndeterminate**()：指示进度条是否在不确定模式下
> - **setIndeterminate**(boolean indeterminate)：设置不确定模式下

#### 2.3.8 SeekBar（拖动条）

是ProgressBar的子类，且含有独特属性**android:thumb**,允许自定义滑块

##### 1.SeekBar基本用法

常用属性，java对应setXxx

> **android:max**="100" //滑动条的最大值
>
> **android:progress**="60" //滑动条的当前值
>
> **android:secondaryProgress**="70" //二级滑动条的进度
>
> **android:thumb** = "@mipmap/sb_icon" //滑块的drawable

SeekBar的事件，**SeekBar.OnSeekBarChangeListener** ,重写三个对应的方法： 

> **onProgressChanged**：进度发生改变时会触发
>
> **onStartTrackingTouch**：按住SeekBar时会触发
>
> **onStopTrackingTouch**：放开SeekBar时触发

#### 2.3.9 RatingBar（星级评分条）

同样是ProgressBar的子类

##### 1.相关属性、事件处理

>  **android:isIndicator**：是否用作指示，用户无法更改，默认false
> **android:numStars**：显示多少个星星，必须为整数
> **android:rating**：默认评分值，必须为浮点数
> **android:stepSize：** 评分每次增加的值，必须为浮点数 

事件处理，只需为RatingBar设置**OnRatingBarChangeListener**事件，然后重写下**onRatingChanged()**方法即可

```java
public class MainActivity extends AppCompatActivity {
    private RatingBar rb_normal;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        rb_normal = (RatingBar) findViewById(R.id.rb_normal);
        rb_normal.setOnRatingBarChangeListener(new RatingBar.OnRatingBarChangeListener() {
            @Override
            public void onRatingChanged(RatingBar ratingBar, float rating, boolean fromUser) {
                Toast.makeText(MainActivity.this, "rating:" + String.valueOf(rating),
                        Toast.LENGTH_LONG).show();
            }
        });
    }
}
```

### 2.4 控件

---

#### 2.4.1 ScrollView(滚动条)

##### 1. 滚动到底部

> 我们可以直接利用ScrollView给我们提供的:**fullScroll()方法**：
>
> scrollView.fullScroll(ScrollView.**FOCUS_DOWN**);滚动到底部
>
> scrollView.fullScroll(ScrollView.**FOCUS_UP**);滚动到顶部
>
> 另外用这玩意的时候要小心异步的玩意，就是addView后，有可能还没有显示完， 如果这个时候直接调用该方法的话，可能会无效，这就需要自己写handler来更新了

##### 2. 设置滚动的滑块图片

> **垂直**方向滑块：android:**scrollbarThumbVertical**
> **水平**方向滑块：android:**scrollbarThumbHorizontal** 

##### 3. 隐藏滑块

> 1.android:scrollbars="none"
> 2.Java代码设置：scrollview.setVerticalScrollBarEnabled(false); 

##### 4. 设置滚动速度

> 这个并没有给我们提供可以直接设置的方法，我们需要自己继承ScrollView，然后重写一个 public void fling (int velocityY)的方法：
>
> ```java
> @Override
> public void fling(int velocityY) {
>     super.fling(velocityY / 2);    //速度变为原来的一半
> }
> ```

#### 2.4.2 Date & Time组件(上)

##### 1. TextClock(文本时钟)

>  TextClock是在Android 4.2(API 17)后推出的用来替代DigitalClock的一个控件！
> TextClock可以以字符串格式显示当前的日期和时间，因此推荐在Android 4.2以后使用TextClock。
> 这个控件推荐在24进制的android系统中使用，TextClock提供了两种不同的格式， 一种是在24进制中显示时间和日期，另一种是在12进制中显示时间和日期。大部分人喜欢默认的设置。 

##### 2. AnalogClock(模拟时钟)

##### 3. Chronometer(计时器)

#### 2.4.3 Date & Time组件(下)

几个原生的Date&Time组件，DatePicker(日期选择器),TimePicker(时间选择器),CalendarView(日期视图)

##### 1. DatePicker(日期选择器)

>- **android:calendarTextColor** ： 日历列表的文本的颜色
>- **android:calendarViewShown**：是否显示日历视图
>- **android:datePickerMode**：组件外观，可选值:spinner，calendar 前者效果如下，默认效果是后者
>-  ![img](https://www.runoob.com/wp-content/uploads/2015/08/47223691.jpg)
>- **android:dayOfWeekBackground**：顶部星期几的背景颜色
>- **android:dayOfWeekTextAppearance**：顶部星期几的文字颜色
>- **android:endYear**：去年(内容)比如2010
>- **android:firstDayOfWeek**：设置日历列表以星期几开头
>- **android:headerBackground**：整个头部的背景颜色
>- **android:headerDayOfMonthTextAppearance**：头部日期字体的颜色
>- **android:headerMonthTextAppearance**：头部月份的字体颜色
>- **android:headerYearTextAppearance**：头部年的字体颜色
>- **android:maxDate**：最大日期显示在这个日历视图mm / dd / yyyy格式
>- **android:minDate**：最小日期显示在这个日历视图mm / dd / yyyy格式
>- **android:spinnersShown**：是否显示spinner
>- **android:startYear**：设置第一年(内容)，比如19940年
>- **android:yearListItemTextAppearance**：列表的文本出现在列表中。
>- **android:yearListSelectorColor**：年列表选择的颜色

##### 2. TimePicker(时间选择器)

>官方提供的属性只有一个：**android:timePickerMode**：组件外观，同样可选值为:spinner和clock(默认) 前者是旧版本的TimePicker~ 而他对应的监听事件是：***TimePicker.OnTimeChangedListener** 

##### 3. CalendarView(日历视图)

> - **android:firstDayOfWeek**：设置一个星期的第一天
> - **android:maxDate** ：最大的日期显示在这个日历视图mm / dd / yyyy格式
> - **android:minDate**：最小的日期显示在这个日历视图mm / dd / yyyy格式
> - **android:weekDayTextAppearance**：工作日的文本出现在日历标题缩写

处理上面的还有其他，但是都是被弃用的... 对应的日期改变事件是：**CalendarView.OnDateChangeListener**

#### 2.4.4 Adaper基础讲解

##### 1. MVC模式的简单理解

ps：吐槽，这个教程的图，其实应该算MVP。不过现在常说的MVC其实就是MVP

##### 2. Adapter概念解析

 ![img](https://www.runoob.com/wp-content/uploads/2015/09/77919389.jpg) 

实际开发常用的几个Adapter

> - **BaseAdapter**：抽象类，实际开发中我们会继承这个类并且重写相关方法，用得最多的一个Adapter！
> - **ArrayAdapter**：支持泛型操作，最简单的一个Adapter，只能展现一行文字~
> - **SimpleAdapter**：同样具有良好扩展性的一个Adapter，可以自定义多种效果！
> - **SimpleCursorAdapter**：用于显示简单文本类型的listView，一般在数据库那里会用到，不过有点过时， 不推荐使用！

基本上用的最多的是BaseAdapter

##### 3. 代码示例:

1. 把数组存储到R.array里的方式，我这里莫名行不通

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //要显示的数据
        String[] strs = {"基神","B神","翔神","曹神","J神"};
        //创建ArrayAdapter
        ArrayAdapter<String> adapter = new ArrayAdapter<String>
                (this,android.R.layout.simple_expandable_list_item_1,strs);
        //获取ListView对象，通过调用setAdapter方法为ListView设置Adapter设置适配器
        ListView list_test = (ListView) findViewById(R.id.list_test);
        list_test.setAdapter(adapter);
    }
}
```

2. ArrayAdapter支持泛型。

3. ArrayAdapter中可用的几种系统自带模板

   >  **simple_list_item_1** *: 单独一行的文本框*
   >
   >  ![img](https://www.runoob.com/wp-content/uploads/2015/09/6830803.jpg) 
   >
   > **simple_list_item_2** *: 两个文本框组成*
   >
   >  ![img](https://www.runoob.com/wp-content/uploads/2015/09/6996906.jpg) 
   >
   > **simple_list_item_checked** *: 每项都是由一个已选中的列表项* ![img](https://www.runoob.com/wp-content/uploads/2015/09/18189803.jpg) 
   >
   > **simple_list_item_multiple_choice** *: 都带有一个复选框* ![img](https://www.runoob.com/wp-content/uploads/2015/09/41311661.jpg) 
   >
   > **simple_list_item_single_choice** *: 都带有一个单选钮* 
   >
   >  ![img](https://www.runoob.com/wp-content/uploads/2015/09/34441475.jpg) 

##### 2. SimpleAdapter使用示例：

先写列表项的xml布局，再java调用

列表项布局文件 **list_item.xml** 

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal">

    <!-- 定义一个用于显示头像的ImageView -->
    <ImageView
        android:id="@+id/imgtou"
        android:layout_width="64dp"
        android:layout_height="64dp"
        android:baselineAlignBottom="true"
        android:paddingLeft="8dp" />

    <!-- 定义一个竖直方向的LinearLayout,把QQ呢称与说说的文本框设置出来 -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView
            android:id="@+id/name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:paddingLeft="8dp"
            android:textColor="#1D1D1C"
            android:textSize="20sp" />

        <TextView
            android:id="@+id/says"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:paddingLeft="8px"
            android:textColor="#B4B4B9"
            android:textSize="14sp" />

    </LinearLayout>

</LinearLayout>
```

MainActivity.java

```java
public class MainActivity extends AppCompatActivity {

    private String[] names = new String[]{"B神", "基神", "曹神"};
    private String[] says = new String[]{"无形被黑，最为致命", "大神好厉害~", "我将带头日狗~"};
    private int[] imgIds = new int[]{R.mipmap.head_icon1, R.mipmap.head_icon2, R.mipmap.head_icon3};

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        List<Map<String, Object>> listitem = new ArrayList<Map<String, Object>>();
        for (int i = 0; i < names.length; i++) {
            Map<String, Object> showitem = new HashMap<String, Object>();
            showitem.put("touxiang", imgIds[i]);
            showitem.put("name", names[i]);
            showitem.put("says", says[i]);
            listitem.add(showitem);
        }

        //创建一个simpleAdapter
        SimpleAdapter myAdapter = new SimpleAdapter(getApplicationContext(), listitem, R.layout.list_item, new String[]{"touxiang", "name", "says"}, new int[]{R.id.imgtou, R.id.name, R.id.says});
        ListView listView = (ListView) findViewById(R.id.list_test);
        listView.setAdapter(myAdapter);
    }
}
```

##### 3. SimpleCursorAdapter使用示例

>  虽然这东西过时了，不过对于不怎么会SQLite的初学者来说，用起来还是蛮方便的！ 记得前面我们学ContentProivder写过的读取联系人的例子么？之前是通过打印Log的 方式显示出来，现在我们通过这个SimpleCursorAdapter把它显示到ListView上 

#### 2.4.5 ListView简单实用

>  *本节我们来继续学习没有讲完的UI控件部分， 回顾上一节，我们介绍了Adapter适配器的概念，然后学习了三个最简单的适配器的使用：*
> *ArrayAdapter，SimpleAdapter和SimpleCursorAdapter，而本节给大家讲解的是第一个 需搭配Adapter使用的UI控件：ListView，不过在版本中被RecyclerView这个新控件替换掉了！*
> *列表作为最常用的控件之一，还是有必要好好学习的，本节以一个初学者的角度来学习 ListView，ListView的属性，以及BaseAdapter简单定义，至于ListView优化这些， 我们一步步来~莫急！* 

##### 1. 自定义BaseAdapter，然后绑定ListView的最简单例子

##### 2. 表头表尾分割线的设置

> listview作为一个列表控件，他和普通的列表一样，可以自己设置表头与表尾： 以及分割线，可供我们设置的属性如下：
>
> - **footerDividersEnabled**：是否在footerView(表尾)前绘制一个分隔条,默认为true
> - **headerDividersEnabled**:是否在headerView(表头)前绘制一个分隔条,默认为true
> - **divider**:设置分隔条,可以用颜色分割,也可以用drawable资源分割
> - **dividerHeight**:设置分隔条的高度
>
> 翻遍了了API发现并没有可以直接设置ListView表头或者表尾的属性，只能在Java中写代码 进行设置了，可供我们调用的方法如下：
>
> - **addHeaderView(View v)**：添加headView(表头),括号中的参数是一个View对象
> - **addFooterView(View v)**：添加footerView(表尾)，括号中的参数是一个View对象
> - **addHeaderView(headView, null, false)**：和前面的区别：设置Header是否可以被选中
> - **addFooterView(View,view,false)**：同上
>
> 对了，使用这个addHeaderView方法必须放在listview.setAdapter前面，否则会报错。

##### 3. 列表从底部开始显示：stackFromBottom

> stackFromBottom属性设置为true

##### 4. 设置点击颜色cacheColorHint

##### 5. 隐藏滑动条

>  *android:scrollbars="none" 或者 setVerticalScrollBarEnabled(true)* 

##### 2.4.6 BaseAdapter优化

>  *上一节中我们学习了如何来使用一个ListView以及自定义一个简单的BaseAdapter，我们从代码 中可以看出比较重要的两个方法:getCount()和getView()，界面上有多少列就会调用多少次getView， 这个时候可能看出一些端倪，每次都是新inflate一个View，都要进行这个XML的解析，这样会 很浪费资源，当然，几十列或者几百列的列表并不能体现什么问题，但假如更多或者布局更加复杂？ 所以学习ListView的优化很重要，而本节针对的是BaseAdapter的优化，优化的两点有，复用convertView 以及使用ViewHolder重用组件，不用每次都findViewById，我们具体通过代码来体会吧！* 

##### 1. 复用ConverView

在BaseAdapter中

public View getView(int position, View convertView, ViewGroup parent) 

的参数convertView是系统提供给我们的可供复用的View的缓存对象，可以先判断是否有值，如果有值就不用再解析xml

```java
@Override
public View getView(int position, View convertView, ViewGroup parent) {

    if(convertView == null){
        convertView = LayoutInflater.from(mContext).inflate(R.layout.item_list_animal,parent,false);
    }

    ImageView img_icon = (ImageView) convertView.findViewById(R.id.img_icon);
    TextView txt_aName = (TextView) convertView.findViewById(R.id.txt_aName);
    TextView txt_aSpeak = (TextView) convertView.findViewById(R.id.txt_aSpeak);

    img_icon.setBackgroundResource(mData.get(position).getaIcon());
    txt_aName.setText(mData.get(position).getaName());
    txt_aSpeak.setText(mData.get(position).getaSpeak());
    return convertView;
}
```

##### 2. ViewHolder重用组件

>  getView()会被调用多次，那么findViewById不一样得调用多次，而我们的ListView的Item 一般都是一样的布局，我们可以对这里在优化下，我们可以自己定义一个ViewHolder类来对这一部分进行性能优化！修改后的代码如下 

```java
@Override
public View getView(int position, View convertView, ViewGroup parent) {
    ViewHolder holder = null;
    if(convertView == null){
        convertView = LayoutInflater.from(mContext).inflate(R.layout.item_list_animal,parent,false);
        holder = new ViewHolder();
        holder.img_icon = (ImageView) convertView.findViewById(R.id.img_icon);
        holder.txt_aName = (TextView) convertView.findViewById(R.id.txt_aName);
        holder.txt_aSpeak = (TextView) convertView.findViewById(R.id.txt_aSpeak);
        convertView.setTag(holder);   //将Holder存储到convertView中
    }else{
        holder = (ViewHolder) convertView.getTag();
    }
    holder.img_icon.setBackgroundResource(mData.get(position).getaIcon());
    holder.txt_aName.setText(mData.get(position).getaName());
    holder.txt_aSpeak.setText(mData.get(position).getaSpeak());
    return convertView;
}

static class ViewHolder{
    ImageView img_icon;
    TextView txt_aName;
    TextView txt_aSpeak;
}
```

>  *没错就是这么简单，你以后BaseAdapter照着这个模板写就对了，哈哈，另外这个修饰ViewHolder的 static，关于是否定义成静态，跟里面的对象数目是没有关系的，加静态是为了在多个地方使用这个 Holder的时候，类只需加载一次，如果只是使用了一次，加不加也没所谓！——***Berial(B神)原话~** 

#### 2.4.7 ListView的焦点问题

-----

> 如果你往ListView的Item中添加了Button，CheckBox，EditText等控件的话，你可能需要考虑 到一个问题：ListView的一个焦点问题！本节我们就来学习下解决这个问题的几个方法！
>
> 我们可以写个简答的listView，上面有一个Button，CheckBox，EditText，但是当我们点击发现， ListView的item点击不了，触发不了onItemClick的方法，也触发不了onItemLongClick方法， 这个就是ListView的一个焦点问题了！就是ListView的焦点被其他控件抢了，下面我们来看看如何 解决这个问题？

##### 方法1：为抢占了控件的组件设置:android:focusable="false"

>  *如题，只需为抢占了ListView Item焦点的控件设置***android:focusable="false"***即可解决这个问题 或者在代码中获得控件后调用：***setFocusable(false)** *!!另外，EditText却不行，如果我们设置了android:focusable="false"，这B可以获取焦点但是一下子 又失去了焦点，而且也不会弹出小键盘，暂不知道如何解决，听别人说是ListView的一个bug，如果 有知道解决方法的欢迎告知下，谢谢~* 

##### 方法2：item根节点设置android:descendantFocusability="blocksDescendants"

> 如题，在Item布局的根节点添加上述属性，**android:descendantFocusability="blocksDescendants"** 即可，另外该属性有三个可供选择的值：
>
> - **beforeDescendants**：viewgroup会优先其子类控件而获取到焦点
> - **afterDescendants**：viewgroup只有当其子类控件不需要获取焦点时才获取焦点
> - **blocksDescendants**：viewgroup会覆盖子类控件而直接获得焦点

#### 2.4.8 ListView之checkbox错位问题解决

##### 1. 问题发生的原因：

 ![img](https://www.runoob.com/wp-content/uploads/2015/09/23101560K-0.jpg) 

 ConvertView会缓存，就是因为这个原因 造成的checkbox错位，所以第一个解决方式就是，不重用这个ConvertView，或者 说每次getView都将这个ConvertView设置为null，但是如果需要显示的Item数目巨大的话， 这种方法就会显得非常臃肿，一般实际开发我们使用的是下面的解决方法： **找个东东来保存当前Item CheckBox的状态，初始化的时候进行判断，设置是否选中** 

##### 2. 解决方法示例

>  *好的存储这个Checkbox的方法有很多，你可以放到一个HashMap中， 每次初始化的时候根据postion取出对应的boolean值，然后再进行checkbox的状态设置； 

  **checkbox监听器的方法要添加在初始化Checkbox状态的代码之前** 

> 找了一篇相关文章，感觉不错
>
>  https://www.cnblogs.com/wujd/archive/2012/08/17/2635309.html 
>
> 个人感觉和Vue、React的组件复用类似，应该是内部处于优化考虑，会对一些缓存的部件重复利用，然而缓冲后的部件只有用户修改的属性会变化，其他的属性还是不变(比如复用了一个组件有个属性check=true,但是程序员以为这是一个新组件，默认check=false,其实是复用了之前的，导致出错)

#### 2.4.9 ListView的数据更新问题

>  *我们前面已经学习了ListView的一些基本用法咧，但是细心的你可能发现了，我们的数据 一开始定义好的，都是静态的，但是实际开发中，我们的数据往往都是动态变化的，比如 我增删该了某一列，那么列表显示的数据也应该进行同步的更新，那么本节我们就来探讨 下ListView数据更新的问题，包括全部更新，以及更新其中的一项，那么开始本节内容* 

重点就是调用 *notifyDataSetChanged()* ，跟新View。

> notifyDataSetChanged()方法会判断是否需要重新渲染，如果当前item没有必要重新渲染 是不会重新渲染的，如果某个Item的状态发生改变，都会导致View的重绘，而重绘的并不是 所有的Item，而是View状态发生变化的那个Item！所以我们直接notifyDataSetChange()方法 即可. 

#### 2.5.0 构建一个可复用的自定义BaseAdapter

> 视频链接
>
>  http://www.imooc.com/learn/372 

#### 2.5.1 ListView Item多布局的实现

>  *这里有个地方要注意的，convertView.setTag(R.id.Tag_APP,holder1);我们平时都直接 setTag(Object)的，这个是setTag的重载方法，参数是一个唯一的key以及后面的一个对象* .

#### 2.5.2 GridView(网格视图)的基本使用

##### 1. 相关属性

> - **android:columnWidth**：设置列的宽度
> - **android:gravity**：组件对其方式
> - **android:horizontalSpacing**：水平方向每个单元格的间距
> - **android:verticalSpacing**：垂直方向每个单元格的间距
> - **android:numColumns**：设置列数
> - **android:stretchMode**：设置拉伸模式，可选值如下： **none**：不拉伸；**spacingWidth**：拉伸元素间的间隔空隙 **columnWidth**：仅仅拉伸表格元素自身 **spacingWidthUniform**：既拉元素间距又拉伸他们之间的间隔空袭

#### 2.5.3 Spinner(列表选项框)的基本使用

##### 1. 相关属性

>- **android:dropDownHorizontalOffset**：设置列表框的水平偏移距离
>- **android:dropDownVerticalOffset**：设置列表框的水平竖直距离
>- **android:dropDownSelector**：列表框被选中时的背景
>- **android:dropDownWidth**：设置下拉列表框的宽度
>- **android:gravity**：设置里面组件的对其方式
>- **android:popupBackground**：设置列表框的背景
>- **android:prompt**：设置对话框模式的列表框的提示信息(标题)，只能够引用string.xml 中的资源id,而不能直接写字符串
>- **android:spinnerMode**：列表框的模式，有两个可选值： **dialog**：对话框风格的窗口 **dropdown**：下拉菜单风格的窗口(默认)
>- 可选属性：**android:entries**：使用数组资源设置下拉列表框的列表项目

##### 2. 使用示例

有一个注意项

> Spinner会默认选中第一个值，就是默认调用spinner.setSection(0), 你可以通过这个设置默认的选中值，另外，会触发一次OnItemSelectedListener 事件，暂时没找到解决方法，下面折衷的处理是：添加一个boolean值，然后设置 为false，在onItemSelected时进行判断，false说明是默认触发的，不做任何操作 将boolean值设置为true；true的话则正常触发事件

#### 2.5.4 AutoCompleteTextView(自动完成文本框)的基本使用

>  *本节继续来学习Adapter类的控件，这次带来的是AutoCompleteTextView(自动完成文本框)， 相信细心的你发现了，和Adapter搭边的控件，都可以自己定义item的样式，是吧！ 或者说每个Item的布局~想怎么玩就怎么玩~嗯，话不多说，开始本节内容~ 对了贴下官方API：*[AutoCompleteTextView](http://androiddoc.qiniudn.com/reference/android/widget/AutoCompleteTextView.html) 

##### 1. 相关属性

> - **android:completionHint**：设置下拉菜单中的提示标题
> - **android:completionHintView**：定义提示视图中显示下拉菜单
> - **android:completionThreshold**：指定用户至少输入多少个字符才会显示提示
> - **android:dropDownAnchor**：设置下拉菜单的定位"锚点"组件，如果没有指定改属性， 将使用该TextView作为定位"锚点"组件
> - **android:dropDownHeight**：设置下拉菜单的高度
> - **android:dropDownWidth**：设置下拉菜单的宽度
> - **android:dropDownHorizontalOffset**：指定下拉菜单与文本之间的水平间距
> - **android:dropDownVerticalOffset**：指定下拉菜单与文本之间的竖直间距
> - **android:dropDownSelector**：设置下拉菜单点击效果
> - **android:popupBackground**：设置下拉菜单的背景

另外其实还有个**MultiAutoCompleteTextView**(多提示项的自动完成文本框) 和这个AutoCompleteTextView作用差不多，属性也一样，具体区别在哪里， 我们在下面的代码中来体验~另外这两个都是全词匹配的，比如，小猪猪： 你输入小->会提示小猪猪，但是输入猪猪->却不会提示小猪猪 

##### 2. 代码示例

部分分析

> 1. android:completionThreshold="1"：这里我们设置了输入一个字就显示提示
> 2. android:completionHint="请输入搜索内容"：这是框框底部显示的文字，如果觉得丑 可以android:completionHintView设置一个View!
> 3. android:dropDownHorizontalOffset="5dp"：设置了水平边距为5dp
> 4. matv_content.setTokenizer(new MultiAutoCompleteTextView.CommaTokenizer()); setTokenizer是为其设置分隔符

#### 2.5.5 ExpandableListView(可折叠列表)的基本使用

##### 1. 相关属性

> - **android:childDivider**：指定各组内子类表项之间的分隔条，图片不会完全显示， 分离子列表项的是一条直线
> - **android:childIndicator**：显示在子列表旁边的Drawable对象，可以是一个图像
> - **android:childIndicatorEnd**：子列表项指示符的结束约束位置
> - **android:childIndicatorLeft**：子列表项指示符的左边约束位置
> - **android:childIndicatorRight**：子列表项指示符的右边约束位置
> - **android:childIndicatorStart**：子列表项指示符的开始约束位置
> - **android:groupIndicator**：显示在组列表旁边的Drawable对象，可以是一个图像
> - **android:indicatorEnd**：组列表项指示器的结束约束位置
> - **android:indicatorLeft**：组列表项指示器的左边约束位置
> - **android:indicatorRight**：组列表项指示器的右边约束位置
> - **android:indicatorStart**：组列表项指示器的开始约束位置

##### 2. 实现ExpandableAdapter的三种方式

> **1.** 扩展**BaseExpandableListAdpter**实现ExpandableAdapter。
>
> **2.** 使用**SimpleExpandableListAdpater**将两个List集合包装成ExpandableAdapter
>
> **3.** 使用**simpleCursorTreeAdapter**将Cursor中的数据包装成SimpleCuroTreeAdapter 本节示例使用的是第一个，扩展BaseExpandableListAdpter，我们需要重写该类中的相关方法， 下面我们通过一个代码示例来体验下！

注意点：

> 核心是重写**BaseExpandableListAdpter**，其实和之前写的普通的BaseAdapter是类似的， 但是BaseExpandableListAdpter则分成了两部分：组和子列表，具体看代码你就知道了！
>
> 另外，有一点要注意的是，重写**isChildSelectable()**方法需要返回true，不然不会触发 子Item的点击事件！

#### 2.5.6 ViewFlipper(翻转视图)的基本使用

>  *本节给大家带了的是ViewFlipper，它是Android自带的一个多页面管理控件，且可以自动播放！ 和ViewPager不同，ViewPager是一页页的，而ViewFlipper则是一层层的，和ViewPager一样，很多时候， 用来实现进入应用后的引导页，或者用于图片轮播，本节我们就使用ViewFlipper写一个简单的图片 轮播的例子吧~官方API：*[ViewFlipper](http://androiddoc.qiniudn.com/reference/android/widget/ViewFlipper.html) 

##### 1. 为ViewFlipper加入View的两种方法

1. 静态导入，也就是XML实现其中每个页面的加载
2. 动态导入，通过java的addView方法填充View

##### 2. 常用的一些方法

> - **setInAnimation**：设置View进入屏幕时使用的动画
> - **setOutAnimation**：设置View退出屏幕时使用的动画
> - **showNext**：调用该方法来显示ViewFlipper里的下一个View
> - **showPrevious**：调用该方法来显示ViewFlipper的上一个View
> - **setFilpInterval**：设置View之间切换的时间间隔
> - **setFlipping**：使用上面设置的时间间隔来开始切换所有的View，切换会循环进行
> - **stopFlipping**：停止View切换

以后做图片轮播和引导页，可以考虑选择用这个实现

#### 2.5.7 Toast(吐司)的基本使用

##### 1. 直接调用Toast类的makeText()方法创建

##### 2. 通过构造方法来定制Toast

1. 定义一个带有图片的Toast
2. Toast完全自定义

#### 2.5.8 Notification(状态栏通知)详解

> 本节带来的是Android中用于在状态栏显示通知信息的控件：Notification，相信大部分 学Android都对他都很熟悉，而网上很多关于Notification的使用教程都是基于2.x的，而 现在普遍的Android设备基本都在4.x以上，甚至是5.0以上的都有；他们各自的Notification 都是不一样的！而本节给大家讲解的是基于4.x以上的Notification，而5.0以上的Notification 我们会在进阶教程的Android 5.0新特性的章节进行讲解~
>
> 官方文档对Notification的一些介绍：
>
> **设计思想**：[Notifications in Android 4.4 and Lower](http://developer.android.com/design/patterns/notifications_k.html)
>
> **译文**：[通知](http://adchs.github.io/patterns/notifications.html)
>
> **API文档**：[Notification](http://developer.android.com/reference/android/app/Notification.html)
>
> 访问上述网站，可能需要梯子哦~

##### 1. 设计文档部分解读

1. Notification的基本布局

    ![img](https://www.runoob.com/wp-content/uploads/2015/09/38056771.jpg) 

   > 上面的组成元素依次是：
   >
   > - **Icon/Photo**：大图标
   > - **Title/Name**：标题
   > - **Message**：内容信息
   > - **Timestamp**：通知时间，默认是系统发出通知的时间，也可以通过setWhen()来设置
   > - **Secondary Icon**：小图标
   > - **内容文字**，在小图标的左手边的一个文字

2. 扩展布局、

##### 2. Notification的基本使用流程

> 状态通知栏主要涉及到2个类：Notification 和NotificationManager
>
> **Notification**：通知信息类，它里面对应了通知栏的各个属性
>
> **NotificationManager**：是状态栏通知的管理类，负责发通知、清除通知等操作。
>
> 使用的基本流程：
>
> - **Step 1.** 获得NotificationManager对象： NotificationManager mNManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
> - **Step 2.** 创建一个通知栏的Builder构造类： Notification.Builder mBuilder = new Notification.Builder(this);
> - **Step 3.** 对Builder进行相关的设置，比如标题，内容，图标，动作等！
> - **Step 4.**调用Builder的build()方法为notification赋值
> - **Step 5**.调用NotificationManager的notify()方法发送通知！
> - **PS:**另外我们还可以调用NotificationManager的cancel()方法取消通知

##### 3. 设置相关的一些方法

> Notification.Builder mBuilder = new Notification.Builder(this);
>
> 后再调用下述的相关的方法进行设置：(官方API文档：[Notification.Builder](http://androiddoc.qiniudn.com/reference/android/app/Notification.Builder.html)) 常用的方法如下：
>
> - **setContentTitle**(CharSequence)：设置标题
>
> - **setContentText**(CharSequence)：设置内容
>
> - **setSubText**(CharSequence)：设置内容下面一小行的文字
>
> - **setTicker**(CharSequence)：设置收到通知时在顶部显示的文字信息
>
> - **setWhen**(long)：设置通知时间，一般设置的是收到通知时的System.currentTimeMillis()
>
> - **setSmallIcon**(int)：设置右下角的小图标，在接收到通知的时候顶部也会显示这个小图标
>
> - **setLargeIcon**(Bitmap)：设置左边的大图标
>
> - **setAutoCancel**(boolean)：用户点击Notification点击面板后是否让通知取消(默认不取消)
>
> - **setDefaults**(int)：向通知添加声音、闪灯和振动效果的最简单、 使用默认（defaults）属性，可以组合多个属性，
>   **Notification.DEFAULT_VIBRATE**(添加默认震动提醒)；
>   **Notification.DEFAULT_SOUND**(添加默认声音提醒)；
>   **Notification.DEFAULT_LIGHTS**(添加默认三色灯提醒)
>   **Notification.DEFAULT_ALL**(添加默认以上3种全部提醒)
>
> - **setVibrate**(long[])：设置振动方式，比如：
>   setVibrate(new long[] {0,300,500,700});延迟0ms，然后振动300ms，在延迟500ms， 接着再振动700ms，关于Vibrate用法后面会讲解！
>
> - **setLights**(int argb, int onMs, int offMs)：设置三色灯，参数依次是：灯光颜色， 亮持续时间，暗的时间，不是所有颜色都可以，这跟设备有关，有些手机还不带三色灯； 另外，还需要为Notification设置flags为Notification.FLAG_SHOW_LIGHTS才支持三色灯提醒！
>
> - **setSound**(Uri)：设置接收到通知时的铃声，可以用系统的，也可以自己设置，例子如下:
>   .setDefaults(Notification.DEFAULT_SOUND) //获取默认铃声
>   .setSound(Uri.parse("file:///sdcard/xx/xx.mp3")) //获取自定义铃声
>   .setSound(Uri.withAppendedPath(Audio.Media.INTERNAL_CONTENT_URI, "5")) //获取Android多媒体库内的铃声
>
> - **setOngoing**(boolean)：设置为ture，表示它为一个正在进行的通知。他们通常是用来表示 一个后台任务,用户积极参与(如播放音乐)或以某种方式正在等待,因此占用设备(如一个文件下载, 同步操作,主动网络连接)
>
> - **setProgress**(int,int,boolean)：设置带进度条的通知 参数依次为：进度条最大数值，当前进度，进度是否不确定 如果为确定的进度条：调用setProgress(max, progress, false)来设置通知， 在更新进度的时候在此发起通知更新progress，并且在下载完成后要移除进度条 ，通过调用setProgress(0, 0, false)既可。如果为不确定（持续活动）的进度条， 这是在处理进度无法准确获知时显示活动正在持续，所以调用setProgress(0, 0, true) ，操作结束时，调用setProgress(0, 0, false)并更新通知以移除指示条
>
> - **setContentIntent**(PendingIntent)：PendingIntent和Intent略有不同，它可以设置执行次数， 主要用于远程服务通信、闹铃、通知、启动器、短信中，在一般情况下用的比较少。比如这里通过 Pending启动Activity：getActivity(Context, int, Intent, int)，当然还可以启动Service或者Broadcast PendingIntent的位标识符(第四个参数)：
>   **FLAG_ONE_SHOT** 表示返回的PendingIntent仅能执行一次，执行完后自动取消
>   **FLAG_NO_CREATE** 表示如果描述的PendingIntent不存在，并不创建相应的PendingIntent，而是返回NULL
>   **FLAG_CANCEL_CURRENT** 表示相应的PendingIntent已经存在，则取消前者，然后创建新的PendingIntent， 这个有利于数据保持为最新的，可以用于即时通信的通信场景
>   **FLAG_UPDATE_CURRENT** 表示更新的PendingIntent
>   使用示例：
>
>   ```
>   //点击后跳转Activity
>   Intent intent = new Intent(context,XXX.class);  
>   PendingIntent pendingIntent = PendingIntent.getActivity(context, 0, intent, 0);  
>   mBuilder.setContentIntent(pendingIntent)  
>   ```
>
>   
>
> - **setPriority**(int)：设置优先级：
>
>   | 优先级  | 用户                                                         |
>   | :------ | :----------------------------------------------------------- |
>   | MAX     | 重要而紧急的通知，通知用户这个事件是时间上紧迫的或者需要立即处理的。 |
>   | HIGH    | 高优先级用于重要的通信内容，例如短消息或者聊天，这些都是对用户来说比较有兴趣的。 |
>   | DEFAULT | 默认优先级用于没有特殊优先级分类的通知。                     |
>   | LOW     | 低优先级可以通知用户但又不是很紧急的事件。                   |
>   | MIN     | 用于后台消息 (例如天气或者位置信息)。最低优先级通知将只在状态栏显示图标，只有用户下拉通知抽屉才能看到内容。 |
>
>   对应属性：Notification.PRIORITY_HIGH...

#### 2.5.9 AlertDialog(对话框)详解

> 本节继续给大家带来是显示提示信息的第三个控件AlertDialog(对话框)，同时它也是其他 Dialog的的父类！比如ProgressDialog，TimePickerDialog等，而AlertDialog的父类是：Dialog！ 另外，不像前面学习的Toast和Notification，AlertDialog并不能直接new出来，如果你打开 AlertDialog的源码，会发现构造方法是protected的，如果我们要创建AlertDialog的话，我们 需要使用到该类中的一个静态内部类：public static class* **Builder**，然后来调用AlertDialog 里的相关方法，来对AlertDialog进行定制，最后调用show()方法来显示我们的AlertDialog对话框！ 好的，下面我们就来学习AlertDialog的基本用法，以及定制我们的AlertDialog！ 官方文档：[AlertDialog](http://androiddoc.qiniudn.com/reference/android/app/AlertDialog.html) 

##### 1. 基本使用流程

> - **Step 1**：创建**AlertDialog.Builder**对象；
> - **Step 2**：调用**setIcon()**设置图标，**setTitle()**或**setCustomTitle()**设置标题；
> - **Step 3**：设置对话框的内容：**setMessage()**还有其他方法来指定显示的内容；
> - **Step 4**：调用**setPositive/Negative/NeutralButton()**设置：确定，取消，中立按钮；
> - **Step 5**：调用**create()**方法创建这个对象，再调用**show()**方法将对话框显示出来；

##### 2. 几种常用的对话框使用示例

##### 3. 通过Builder的setView()定制显示的AlertDialog

#### 2.6.0 其他几种常用对话框基本使用

##### 1. ProgressDialog(进度条对话框)的基本使用

> 我们创建进度条对话框的方式有两种：
>
> - **1**.直接调用ProgressDialog提供的静态方法show()显示
> - **2**.创建ProgressDialog,再设置对话框的参数,最后show()出来

##### 2. DatePickerDialog(日期选择对话框)与TimePickerDialog(时间选择对话框)

> 先要说明一点： Date/TimePickerDialog只是供用户来选择日期时间,对于android系统的系统时间, 日期没有任何影响,google暂时没有公布系统日期时间设置的API, 如果要在app中设置的话,要重新编译android的系统源码，非常麻烦！
>
> 他们两个的构造方法非常相似： **DatePickerDialog**(上下文；DatePickerDialog.OnDateSetListener()监听器；年；月；日)
> **TimePickerDialog**(上下文；TimePickerDialog.OnTimeSetListener()监听器；小时，分钟，是否采用24小时制)

#### 2.6.1 PopupWindow(悬浮框)的基本使用

> 本节给大家带来的是最后一个用于显示信息的UI控件——PopupWindow(悬浮框)，如果你想知道 他长什么样子，你可以打开你手机的QQ，长按列表中的某项，这个时候后弹出一个黑色的小 对话框，这种就是PopupWindow了，和AlertDialog对话框不同的是，他的位置可以是随意的；
>
> 另外AlertDialog是非堵塞线程的，而PopupWindow则是堵塞线程的！而官方有这样一句话来介绍 PopupWindow：
>
> **A popup window that can be used to display an arbitrary view. The popup window is**
>
> **a floating container that appears on top of the current activity.**
>
> 大概意思是：一个弹出窗口控件，可以用来显示任意View，而且会浮动在当前activity的顶部

注意点：PopupWindow是线程阻塞的；AlertDialog是非线程阻塞的。

##### 1. 相关方法的解读

1. 几个常用的构造方法

   > 我们在文档中可以看到，提供给我们的PopupWindow的构造方法有九种之多，这里只贴实际 开发中用得较多的几个构造方法：
   >
   > - **public PopupWindow (Context context)**
   > - **public PopupWindow(View contentView, int width, int height)**
   > - **public PopupWindow(View contentView)**
   > - **public PopupWindow(View contentView, int width, int height, boolean focusable)**
   >
   > 参数就不用多解释了吧，contentView是PopupWindow显示的View，focusable是否显示焦点

2. 常用的一些方法

   > 下面介绍几个用得较多的一些方法，其他的可自行查阅文档：
   >
   > - **setContentView**(View contentView)：设置PopupWindow显示的View
   > - **getContentView**()：获得PopupWindow显示的View
   > - **showAsDropDown(View anchor)**：相对某个控件的位置（正左下方），无偏移
   > - **showAsDropDown(View anchor, int xoff, int yoff)**：相对某个控件的位置，有偏移
   > - **showAtLocation(View parent, int gravity, int x, int y)**： 相对于父控件的位置（例如正中央Gravity.CENTER，下方Gravity.BOTTOM等），可以设置偏移或无偏移 PS:parent这个参数只要是activity中的view就可以了！
   > - **setWidth/setHeight**：设置宽高，也可以在构造方法那里指定好宽高， 除了可以写具体的值，还可以用WRAP_CONTENT或MATCH_PARENT， popupWindow的width和height属性直接和第一层View相对应。
   > - **setFocusable(true)**：设置焦点，PopupWindow弹出后，所有的触屏和物理按键都由PopupWindows 处理。其他任何事件的响应都必须发生在PopupWindow消失之后，（home 等系统层面的事件除外）。 比如这样一个PopupWindow出现的时候，按back键首先是让PopupWindow消失，第二次按才是退出 activity，准确的说是想退出activity你得首先让PopupWindow消失，因为不并是任何情况下按back PopupWindow都会消失，必须在PopupWindow设置了背景的情况下 。
   > - **setAnimationStyle(int)：**设置动画效果

##### 2.6.2 菜单(Menu)

> 本章给大家带来的是Android中的Menu(菜单)，而在Android中的菜单有如下几种：
>
> - **OptionMenu**：选项菜单，android中最常见的菜单，通过Menu键来调用
> - **SubMenu**：子菜单，android中点击子菜单将弹出一个显示子菜单项的悬浮框， 子菜单不支持嵌套，即不能包括其他子菜单
> - **ContextMenu**：上下文菜单，通过长按某个视图组件后出现的菜单，该组件需注册上下文菜单 本节我们来依依学习这几种菜单的用法~ PS：官方文档：[menus](http://androiddoc.qiniudn.com/guide/topics/ui/menus.html)

##### 1. OptionMenu(选项菜单)

> 答：非常简单，重写两个方法就好，其实这两个方法我们在创建项目的时候就会自动生成~ 他们分别是：
>
> - public boolean **onCreateOptionsMenu**(Menu menu)：调用OptionMenu，在这里完成菜单初始化
> - public boolean **onOptionsItemSelected**(MenuItem item)：菜单项被选中时触发，这里完成事件处理
>
> 当然除了上面这两个方法我们可以重写外我们还可以重写这三个方法：
>
> - public void **onOptionsMenuClosed**(Menu menu)：菜单关闭会调用该方法
> - public boolean **onPrepareOptionsMenu**(Menu menu)：选项菜单显示前会调用该方法， 可在这里进行菜单的调整(动态加载菜单列表)
> - public boolean **onMenuOpened**(int featureId, Menu menu)：选项菜单打开以后会调用这个方法

 而加载菜单的方式有两种，一种是直接通过编写菜单XML文件，然后调用： **getMenuInflater().inflate(R.menu.menu_main, menu);**加载菜单 或者通过代码动态添加，onCreateOptionsMenu的参数menu，调用add方法添加 菜单，add(菜单项的组号，ID，排序号，标题)，<u>另外如果排序号是按添加顺序排序的话都填0即可</u>！ 

##### 2. ContextMenu(上下文菜单)

长按某个View后会出现的View，我们需要为这个View注册上下文菜单

> - **Step 1**：重写onCreateContextMenu()方法
> - **Step 2**：为view组件注册上下文菜单，使用registerForContextMenu()方法,参数是View
> - **Step 3**：重写onContextItemSelected()方法为菜单项指定事件监听器

建议使用XML实现菜单项

##### 3. SubMenu(子菜单)

就是在\<item\>里多嵌套一层\<menu\>

##### 4. PopupMenu(弹出式菜单)

 一个类似于PopupWindow的东东，他可以很方便的在指定View下显示一个弹出菜单，而且 他的菜单选项可以来自于Menu资源 

#### 2.6.3 ViewPager的简单使用

>  *本节带来的是Android 3.0后引入的一个UI控件——ViewPager(视图滑动切换工具)，实在想不到 如何来称呼这个控件，他的大概功能：通过手势滑动可以完成View的切换，一般是用来做APP 的引导页或者实现图片轮播，因为是3.0后引入的，如果想在低版本下使用，就需要引入v4 兼容包哦~，我们也可以看到，ViewPager在：android.support.v4.view.ViewPager目录下~ 下面我们就来学习一下这个控件的基本用法~ 官方API文档：*[ViewPager](http://androiddoc.qiniudn.com/reference/android/support/v4/view/ViewPager.html) 

##### 1. ViewPager的简单介绍

> ViewPager就是一个简单的页面切换组件，我们可以往里面填充多个View，然后我们可以左 右滑动，从而切换不同的View，我们可以通过setPageTransformer()方法为我们的ViewPager 设置切换时的动画效果，当然，动画我们还没学到，所以我们把为ViewPager设置动画 放到下一章绘图与动画来讲解！和前面学的ListView，GridView一样，我们也需要一个Adapter (适配器)将我们的View和ViewPager进行绑定，而ViewPager则有一个特定的Adapter—— **PagerAdapter**！另外，Google官方是建议我们使用Fragment来填充ViewPager的，这样 可以更加方便的生成每个Page，以及管理每个Page的生命周期！给我们提供了两个Fragment 专用的Adapter：**FragmentPageAdapter**和**FragmentStatePagerAdapter** 我们简要的来分析下这两个Adapter的区别：
>
> - **FragmentPageAdapter**：和PagerAdapter一样，只会缓存当前的Fragment以及左边一个，右边 一个，即总共会缓存3个Fragment而已，假如有1，2，3，4四个页面：
>   处于1页面：缓存1，2
>   处于2页面：缓存1，2，3
>   处于3页面：销毁1页面，缓存2，3，4
>   处于4页面：销毁2页面，缓存3，4
>   更多页面的情况，依次类推~
> - **FragmentStatePagerAdapter**：当Fragment对用户不 见得时，整个Fragment会被销毁， 只会保存Fragment的状态！而在页面需要重新显示的时候，会生成新的页面！
>
> 综上，FragmentPageAdapter适合固定的页面较少的场合；而FragmentStatePagerAdapter则适合 于页面较多或者页面内容非常复杂(需占用大量内存)的情况！

##### 2. PagerAdapter的使用

> 我们先来介绍最普通的PagerAdapter，如果想使用这个PagerAdapter需要重写下面的四个方法： 当然，这只是官方建议，实际上我们只需重写getCount()和isViewFromObject()就可以了~
>
> - **getCount()**:获得viewpager中有多少个view
> - **destroyItem()**:移除一个给定位置的页面。适配器有责任从容器中删除这个视图。 这是为了确保在finishUpdate(viewGroup)返回时视图能够被移除。
>
> 而另外两个方法则是涉及到一个key的东东：
>
> - **instantiateItem()**: ①将给定位置的view添加到ViewGroup(容器)中,创建并显示出来 ②返回一个代表新增页面的Object(key),通常都是直接返回view本身就可以了,当然你也可以 自定义自己的key,但是key和每个view要一一对应的关系
> - **isViewFromObject()**: 判断instantiateItem(ViewGroup, int)函数所返回来的Key与一个页面视图是否是 代表的同一个视图(即它俩是否是对应的，对应的表示同一个View),通常我们直接写 return view == object!

1. 最简单用法

   就简单的切换设定的视图

2. 标题栏——PagerTitleStrip与PagerTabStrip

    这里两者的区别仅仅是布局不一样而已，其他的都一样

   还需要重写getPageTitle(),设置标题

3.  ViewPager实现TabHost效果

   > 首先是布局：顶部一个LinearLayout，包着三个TextView，weight属性都为1，然后下面跟着 一个滑块的ImageView，我们设置宽度为match_parent；最底下是我们的ViewPager，这里可能 有两个属性你并不认识，一个是：flipInterval：这个是指定View动画间的时间间隔的！
   > 而persistentDrawingCache：则是设置控件的绘制缓存策略，可选值有四个：
   >
   > - none：不在内存中保存绘图缓存；
   > - animation:只保存动画绘图缓存；
   > - scrolling：只保存滚动效果绘图缓存；
   > - all：所有的绘图缓存都应该保存在内存中；
   >
   > 可以同时用2个，animation|scrolling这样

#### 2.6.4 DrawerLayout(官方侧滑菜单)的简单使用

##### 1. 使用的注意事项

> - **1**.主内容视图一定要是DrawerLayout的第一个子视图
> - **2**.主内容视图宽度和高度需要match_parent
> - **3**.必须显示指定侧滑视图的android:**layout_gravity属性** android:layout_gravity = "start"时，从左向右滑出菜单 android:layout_gravity = "end"时，从右向左滑出菜单 不推荐使用left和right!!!
> - 侧滑视图的宽度以dp为单位，不建议超过**320dp**(为了总能看到一些主内容视图)
> - 设置侧滑事件：mDrawerLayout.setDrawerListener(DrawerLayout.DrawerListener);
> - 要说一点：可以结合Actionbar使用当用户点击Actionbar上的应用图标，弹出侧滑菜单！ 这里就要通过**ActionBarDrawerToggle**，它是DrawerLayout.DrawerListener的具体实现类， 我们可以重写ActionBarDrawerToggle的onDrawerOpened()和onDrawerClosed()以监听抽屉拉出 或隐藏事件！但是这里我们不讲，因为5.0后我们使用的是Toolbar！有兴趣的可以自行查阅相关 文档！

可能有的疑惑

> - **1**. drawer_layout.**openDrawer**(Gravity.END);
>   这句是设置打开的哪个菜单START代表左边，END代表右边
> - **2**. drawer_layout.setDrawerLockMode(DrawerLayout.LOCK_MODE_LOCKED_CLOSED,Gravity.END); 锁定右面的侧滑菜单，不能通过手势关闭或者打开，只能通过代码打开！即调用openDrawer方法！ 接着 drawer_layout.setDrawerLockMode(DrawerLayout.LOCK_MODE_UNLOCKED,Gravity.END); 解除锁定状态，即可以通过手势关闭侧滑菜单 最后在drawer关闭的时候调用： drawer_layout.setDrawerLockMode(DrawerLayout.LOCK_MODE_LOCKED_CLOSED, Gravity.END); 再次锁定右边的侧滑菜单！
> - **3**. 布局代码中的Tag属性的作用？ 答：这里没用到，在重写DrawerListener的onDrawerSlide方法时，我们可以通过他的第一个 参数drawerView，调用drawerView.getTag().equals("START")判断触发菜单事件的是哪个 菜单！然后可以进行对应的操作！

### 第三章--Android的事件处理机制

---

#### 3.1.1 基于监听的事件处理机制

###### 1. 基于监听的事件处理机制模型

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/4109430.jpg) 

##### 2. 五种不同的使用形式

1. 使用匿名内部类(常用)
2. 使用内部类
3. 使用外部类
4. 直接使用Activity作为事件监听器

5. 直接绑定导标签(也就是XML上指定onClick="java中写的方法")

#### 3.2 基于回调的事件处理机制

##### 1. 什么是方法回调

个人感觉指的就是各种框架之类的常带钩子函数、回调函数。也就是在一些常见的应用场景定义一些方法，这些方法会在某场景结束或开始或执行中被执行，而我们只需要重写这些方法，就可以在指定的时刻/位置执行我们想要的操作(其实就是AOP的具体应用)

##### 2. Android回调的事件处理机制详解

1.  自定义View

   >  当用户在GUI组件上激发某个事件时,组件有自己特定的方法会负责处理该事件 通常用法:继承基本的GUI组件,重写该组件的事件处理方法,即自定义view 注意:在xml布局中使用自定义的view时,需要使用**"全限定类名"**

   常见的View组件的回调方法

   > ①在该组件上触发屏幕事件: boolean onTouchEvent(MotionEvent event);
   > ②在该组件上按下某个按钮时: boolean onKeyDown(int keyCode,KeyEvent event);
   > ③松开组件上的某个按钮时: boolean onKeyUp(int keyCode,KeyEvent event);
   > ④长按组件某个按钮时: boolean onKeyLongPress(int keyCode,KeyEvent event);
   > ⑤键盘快捷键事件发生: boolean onKeyShortcut(int keyCode,KeyEvent event);
   > ⑥在组件上触发轨迹球屏事件: boolean onTrackballEvent(MotionEvent event);
   > ⑦当组件的焦点发生改变,和前面的6个不同,这个方法只能够在View中重写哦！ protected void onFocusChanged(boolean gainFocus, int direction, Rect previously FocusedRect)

2. 基于回调的事件传播

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/9989678.jpg) 

注意点就是，弄清设置监听器、重写组件事件触发回调方法以及Activity的事件触发回调方法的执行顺序。传播的顺序是: **监听器**--->**view组件的回调方法**--->**Activity的回调方法**

#### 3.3 Handler消息传递机制浅析

>  前两节中我们对Android中的两种事件处理机制进行了学习，关于响应的事件响应就这两种；本节给大家讲解的 是Activity中UI组件中的信息传递Handler，相信很多朋友都知道，Android为了线程安全，并不允许我们在UI线程外操作UI；很多时候我们做界面刷新都需要通过Handler来通知UI组件更新！除了用Handler完成界面更新外，还可以使用runOnUiThread()来更新，甚至更高级的事务总线，当然，这里我们只讲解Handler，什么是Handler，执行流程，相关方法，子线程与主线程中中使用Handler的区别 

##### 1. 学习路线

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/70402782.jpg) 

##### 2. Handler类的引入

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/90456225.jpg) 

##### 3. Handler的执行流程图

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/25345060.jpg) 

相关名词：

> - **UI线程**:就是我们的主线程,系统在创建UI线程的时候会初始化一个Looper对象,同时也会创建一个与其关联的MessageQueue;
> - **Handler**:作用就是发送与处理信息,如果希望Handler正常工作,在当前线程中要有一个Looper对象
> - **Message**:Handler接收与处理的消息对象
> - **MessageQueue**:消息队列,先进先出管理Message,在初始化Looper对象时会创建一个与之关联的MessageQueue;
> - **Looper**:每个线程只能够有一个Looper,管理MessageQueue,不断地从中取出Message分发给对应的Handler处理！

简言之：

>  *当我们的子线程想修改Activity中的UI组件时,我们可以新建一个Handler对象,通过这个对象向主线程发送信息;而我们发送的信息会先到主线程的MessageQueue进行等待,由Looper按先入先出顺序取出,再根据message对象的what属性分发给对应的Handler进行处理* 

##### 4. Handler的相关方法

> - void **handleMessage**(Message msg):处理消息的方法,通常是用于被重写!
> - **sendEmptyMessage**(int what):发送空消息
> - **sendEmptyMessageDelayed**(int what,long delayMillis):指定延时多少毫秒后发送空信息
> - **sendMessage**(Message msg):立即发送信息
> - **sendMessageDelayed**(Message msg):指定延时多少毫秒后发送信息
> - final boolean **hasMessage**(int what):检查消息队列中是否包含what属性为指定值的消息 如果是参数为(int what,Object object):除了判断what属性,还需要判断Object属性是否为指定对象的消息

##### 5. Handler的使用示例

1. Handler写在主线程中(这个感觉一般用不到，一般都是子线程需要更新UI才使用Handler)

   >  *在主线程中,因为系统已经初始化了一个Looper对象,所以我们直接创建Handler对象,就可以进行信息的发送与处理了* 

2. Handler写在子线程中

    如果是Handler写在了子线程中的话,我们就需要自己创建一个Looper对象：

   >  1 )直接调用Looper.prepare()方法即可为当前线程创建Looper对象,而它的构造器会创建配套的MessageQueue;
   > 2 )创建Handler对象,重写handleMessage( )方法就可以处理来自于其他线程的信息了!
   > 3 )调用Looper.loop()方法启动Looper

ps:感觉这里举的例子比较复杂，之前用到的时候不过就是Handler通知主线程更新UI，这里安利几篇相关文章：

> 异步加载资源经常需要渲染UI，但是只能主线程渲染UI，这时候就需要Handler了
>
> [Android基础夯实--你了解Handler有多少？]( https://www.cnblogs.com/ryanleee/p/8204450.html )
>
> [Android多线程：手把手教你使用HandlerThread]( https://www.jianshu.com/p/9c10beaa1c95 )

看了网上的几篇文章后，总结一下思路。主线程(UI线程)本来初始化就带有Looper，而Looper是绑定所对应的Thread的，其管理一个MessageQueue。创建Handler需要设定其对应的Looper，这样才能知道这个Handler该把消息给谁，或者处理谁的消息。一般使用的情景都是子线程需要渲染UI，但是Andorid考虑到安全、同步的问题，设定只有主线程(UI线程)可以渲染UI，所以我们一般新建一个Handler并将主线程的Looper绑定到子线程，这样子就可以在子线程做好资源加载后，反馈给主线程的Looper，在主线程消息队列MessageQueue轮到该Handler发出的消息执行时，就可以渲染子线程要求的资源到UI界面上了。

#### 3.4.1 TouchListener PK OnTouchEvent + 多点触碰

>  *TouchListener是基于监听的，而OnTouchEvent则是基于回调的* 

##### 1. 基于监听的TouchListener

OnTouchListener相关方法与属性

> onTouch(View v, MotionEvent event):这里面的参数依次是触发触摸事件的组件,触碰事件event 封装了触发事件的详细信息，同样包括事件的类型、触发时间等信息。比如event.getX(),event.getY()
> 我们也可以对触摸的动作类型进行判断,使用event.getAction( )再进行判断;如:
> event.getAction == MotionEvent.ACTION_DOWN：按下事件
> event.getAction == MotionEvent.ACTION_MOVE:移动事件
> event.getAction == MotionEvent.ACTION_UP:弹起事件

##### 2. 基于回调的onTouchEvent( )方法

> 同样是触碰事件,但是**onTouchEvent更多的是用于自定义的view**,所有的view类中都重写了该方法,而这种触摸事件是基于回调的,也就是说:如果我们返回的值是false的话,那么事件会继续向外传播,由外面的容器或者Activity进行处理!当然还涉及到了手势(Gesture),这个我们会在后面进行详细的讲解!**onTouchEvent其实和onTouchListener是类似的,只是处理机制不同,前者是回调,后者是监听模式**

主要就是Paint、onDraw(Canvas canvas)、invalidate()这几个点需要注意

下面是示例的自定义View代码

```java 
public class MyView extends View{  
    public float X = 50;  
    public float Y = 50;  
  
    //创建画笔  
    Paint paint = new Paint();  
  
    public MyView(Context context,AttributeSet set)  
    {  
        super(context,set);  
    }  
  
    @Override  
    public void onDraw(Canvas canvas) {  
        super.onDraw(canvas);  
        paint.setColor(Color.BLUE);  
        canvas.drawCircle(X,Y,30,paint);  
    }  
  
    @Override  
    public boolean onTouchEvent(MotionEvent event) {  
        this.X = event.getX();  
        this.Y = event.getY();  
        //通知组件进行重绘  
        this.invalidate();  
        return true;  
    }  
}
```

##### 3. 多点触碰

原理类的东西：

所谓的多点触碰就是多个手指在屏幕上进行操作，用的最多的估计是放大缩功能吧，比如很多的图片浏览器都支持缩放！理论上Android系统本身可以处理多达256个手指的触摸，当然这取决于手机硬件的支持；不过支持多点触摸的手机一般支持2-4个点，当然有些更多！

我们发现前面两点都有用到MotionEvent，MotionEvent代表的是一个触摸事件；

我们可以根据**event.getAction() & MotionEvent.ACTION_MASK**来判断是哪种操作，除了上面介绍的三种单点操作外，还有两个多点专用的操作：

- MotionEvent.**ACTION_POINTER_DOWN**:当屏幕上已经有一个点被按住，此时再按下其他点时触发。
- MotionEvent.**ACTION_POINTER_UP**:当屏幕上有多个点被按住，松开其中一个点时触发（即非最后一个点被放开时）。

**简单流程大致如下**：

- 当我们一个手指触摸屏幕 ——> 触发ACTION_DOWN事件
- 接着有另一个手指也触摸屏幕 ——> 触发ACTION_POINTER_DOWN事件,如果还有其他手指触摸，继续触发
- 有一个手指离开屏幕 ——> 触发ACTION_POINTER_UP事件，继续有手指离开，继续触发
- 当最后一个手指离开屏幕 ——> 触发ACTION_UP事件
- **而且在整个过程中，ACTION_MOVE事件会一直不停地被触发**

> 我们可以通过event.**getX**(int)或者event.**getY**(int)来获得不同触摸点的位置： 比如event.getX(0)可以获得第一个接触点的X坐标，event.getX(1)获得第二个接触点的X坐标这样... 另外，我们还可以在调用MotionEvent对象的**getPointerCount()**方法判断当前有多少个手指在触摸 

java部分的代码：

```java
package com.jay.example.edittextdemo;

import android.app.Activity;
import android.graphics.Matrix;
import android.graphics.PointF;
import android.os.Bundle;
import android.util.FloatMath;
import android.view.MotionEvent;
import android.view.View;
import android.view.View.OnTouchListener;
import android.widget.ImageView;

public class MainActivity extends Activity implements OnTouchListener {

    private ImageView img_test;

    // 縮放控制
    private Matrix matrix = new Matrix();
    private Matrix savedMatrix = new Matrix();

    // 不同状态的表示：
    private static final int NONE = 0;
    private static final int DRAG = 1;
    private static final int ZOOM = 2;
    private int mode = NONE;

    // 定义第一个按下的点，两只接触点的重点，以及出事的两指按下的距离：
    private PointF startPoint = new PointF();
    private PointF midPoint = new PointF();
    private float oriDis = 1f;

    /*
     * (non-Javadoc)
     * 
     * @see android.app.Activity#onCreate(android.os.Bundle)
     */
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        img_test = (ImageView) this.findViewById(R.id.img_test);
        img_test.setOnTouchListener(this);
    }

    @Override
    public boolean onTouch(View v, MotionEvent event) {
        ImageView view = (ImageView) v;
        switch (event.getAction() & MotionEvent.ACTION_MASK) {
        // 单指
        case MotionEvent.ACTION_DOWN:
            matrix.set(view.getImageMatrix());
            savedMatrix.set(matrix);
            startPoint.set(event.getX(), event.getY());
            mode = DRAG;
            break;
        // 双指
        case MotionEvent.ACTION_POINTER_DOWN:
            oriDis = distance(event);
            if (oriDis > 10f) {
                savedMatrix.set(matrix);
                midPoint = middle(event);
                mode = ZOOM;
            }
            break;
        // 手指放开
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_POINTER_UP:
            mode = NONE;
            break;
        // 单指滑动事件
        case MotionEvent.ACTION_MOVE:
            if (mode == DRAG) {
                // 是一个手指拖动
                matrix.set(savedMatrix);
                matrix.postTranslate(event.getX() - startPoint.x, event.getY() - startPoint.y);
            } else if (mode == ZOOM) {
                // 两个手指滑动
                float newDist = distance(event);
                if (newDist > 10f) {
                    matrix.set(savedMatrix);
                    float scale = newDist / oriDis;
                    matrix.postScale(scale, scale, midPoint.x, midPoint.y);
                }
            }
            break;
        }
        // 设置ImageView的Matrix
        view.setImageMatrix(matrix);
        return true;
    }

    // 计算两个触摸点之间的距离
    private float distance(MotionEvent event) {
        float x = event.getX(0) - event.getX(1);
        float y = event.getY(0) - event.getY(1);
        return FloatMath.sqrt(x * x + y * y);
    }

    // 计算两个触摸点的中点
    private PointF middle(MotionEvent event) {
        float x = event.getX(0) + event.getX(1);
        float y = event.getY(0) + event.getY(1);
        return new PointF(x / 2, y / 2);
    }

}
```

重点就是知道图像是用matrix矩阵表示的，然后用以下2个方法分别控制图片移动和缩放

+ 图片移动

  **matrix.postTranslate(event.getX() - startPoint.x, event.getY() - startPoint.y);**

+ 图片缩放

  **matrix.postScale(scale, scale, midPoint.x, midPoint.y);**

> 这里贴一个官方Matrix的API介绍
>
>  https://www.android-doc.com/reference/android/graphics/Matrix.html 

#### 3.5 监听EditText的内容变化

>  在前面我们已经学过EditText控件了，本节来说下如何监听输入框的内容变化！ 这个再实际开发中非常实用，另外，附带着说下如何实现EditText的密码可见 与不可见 

##### 1. 监听EditText的内容变化

>  *由题可知，是基于监听的事件处理机制，好像前面的点击事件是OnClickListener，文本内容 变化的监听器则是：TextWatcher，我们可以调用EditText.addTextChangedListener(mTextWatcher); 为EditText设置内容变化监听* 

 简单说下TextWatcher，实现该类需实现三个方法： 

```java
public void beforeTextChanged(CharSequence s, int start,int count, int after);   
public void onTextChanged(CharSequence s, int start, int before, int count);
public void afterTextChanged(Editable s);
```

依次会在下述情况中触发：

- 1.内容变化前
- 2.内容变化中
- 3.内容变化后（最常用）

我们可以根据实际的需求重写相关方法，一般重写得较多的是第三个方法！

监听EditText内容变化的场合有很多： 限制字数输入，限制输入内容等等~

这里给大家实现一个简单的自定义EditText，输入内容后，有面会显示一个叉叉的圆圈，用户点击后 可以清空文本框~，当然你也可以不自定义，直接为EditText添加TextWatcher然后设置下删除按钮~

##### 2. 实现EditText的密码可见与不可见

#### 3.6 响应系统设置的事件(Configuration类)

>  本节给大家介绍的Configuration类是用来描述手机设备的配置信息的，比如屏幕方向， 触摸屏的触摸方式等，相信定制过ROM的朋友都应该知道我们可以在: frameworks/base/core/java/android/content/res/Configuration.java 找到这个类，然后改下相关设置，比如调整默认字体的大小！有兴趣可自行了解！ 本节讲解的Configuration类在我们Android开发中的使用~ API文档：[Configuration](http://androiddoc.qiniudn.com/reference/android/content/res/Configuration.html) 

##### 1. Configuration给我们提供的方法列表

> - **densityDpi**：屏幕密度
> - **fontScale**：当前用户设置的字体的缩放因子
> - **hardKeyboardHidden**：判断硬键盘是否可见，有两个可选值：HARDKEYBOARDHIDDEN_NO,HARDKEYBOARDHIDDEN_YES，分别是十六进制的0和1
> - **keyboard**：获取当前关联额键盘类型：该属性的返回值：KEYBOARD_12KEY（只有12个键的小键盘）、KEYBOARD_NOKEYS、KEYBOARD_QWERTY（普通键盘）
> - **keyboardHidden**：该属性返回一个boolean值用于标识当前键盘是否可用。该属性不仅会判断系统的硬件键盘，也会判断系统的软键盘（位于屏幕）。
> - **locale**：获取用户当前的语言环境
> - **mcc**：获取移动信号的国家码
> - **mnc**：获取移动信号的网络码
>   ps:国家代码和网络代码共同确定当前手机网络运营商
> - **navigation**：判断系统上方向导航设备的类型。该属性的返回值：NAVIGATION_NONAV（无导航）、 NAVIGATION_DPAD(DPAD导航）NAVIGATION_TRACKBALL（轨迹球导航）、NAVIGATION_WHEEL（滚轮导航）
> - **orientation**：获取系统屏幕的方向。该属性的返回值：ORIENTATION_LANDSCAPE（横向屏幕）、ORIENTATION_PORTRAIT（竖向屏幕）
> - **screenHeightDp**，**screenWidthDp**：屏幕可用高和宽，用dp表示
> - **touchscreen**：获取系统触摸屏的触摸方式。该属性的返回值：TOUCHSCREEN_NOTOUCH（无触摸屏）、TOUCHSCREEN_STYLUS（触摸笔式触摸屏）、TOUCHSCREEN_FINGER（接收手指的触摸屏）

##### 2.示例代码:

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView txtResult = (TextView) findViewById(R.id.txtResult);
        StringBuffer status = new StringBuffer();
        //①获取系统的Configuration对象
        Configuration cfg = getResources().getConfiguration();
        //②想查什么查什么
        status.append("densityDpi:" + cfg.densityDpi + "\n");
        status.append("fontScale:" + cfg.fontScale + "\n");
        status.append("hardKeyboardHidden:" + cfg.hardKeyboardHidden + "\n");
        status.append("keyboard:" + cfg.keyboard + "\n");
        status.append("keyboardHidden:" + cfg.keyboardHidden + "\n");
        status.append("locale:" + cfg.locale + "\n");
        status.append("mcc:" + cfg.mcc + "\n");
        status.append("mnc:" + cfg.mnc + "\n");
        status.append("navigation:" + cfg.navigation + "\n");
        status.append("navigationHidden:" + cfg.navigationHidden + "\n");
        status.append("orientation:" + cfg.orientation + "\n");
        status.append("screenHeightDp:" + cfg.screenHeightDp + "\n");
        status.append("screenWidthDp:" + cfg.screenWidthDp + "\n");
        status.append("screenLayout:" + cfg.screenLayout + "\n");
        status.append("smallestScreenWidthDp:" + cfg.densityDpi + "\n");
        status.append("touchscreen:" + cfg.densityDpi + "\n");
        status.append("uiMode:" + cfg.densityDpi + "\n");
        txtResult.setText(status.toString());
    }
}
```

##### 3.重写onConfigurationChanged响应系统设置更改

>  *该方法用于监听系统设置的更改,是基于回调的时间处理方法,当系统的设置发生改变时就会自动触发; 但是要注意一点,使用下面的方法监控的话,targetSdkVersion属性最高只能设置为12,高于12的话,该方法不会被激发！这里写个横竖屏切换的例子给大家参考参考，其他的可自行谷歌* 

实现代码：

```java
public class MainActivity extends Activity {  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
          
        Button btn = (Button) findViewById(R.id.btncahange);  
        btn.setOnClickListener(new OnClickListener() {  
              
            @Override  
            public void onClick(View v) {  
                Configuration config = getResources().getConfiguration();  
                //如果是横屏的话切换成竖屏  
                if(config.orientation == Configuration.ORIENTATION_LANDSCAPE)  
                {  
                    MainActivity.this.setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);  
                }  
                //如果竖屏的话切换成横屏  
                if(config.orientation == Configuration.ORIENTATION_PORTRAIT)  
                {  
                    MainActivity.this.setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);  
                }  
            }  
        });  
    }  
      
    @Override  
    public void onConfigurationChanged(Configuration newConfig) {  
        super.onConfigurationChanged(newConfig);  
        String screen = newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE?"横屏":"竖屏";  
        Toast.makeText(MainActivity.this, "系统屏幕方向发生改变 \n 修改后的方向为" + screen, Toast.LENGTH_SHORT).show();  
    }  
} 
```

 另外，还需要在AndroidManifest.xml添加下述内容： 

>  *权限:* **< uses-permission android:name="android.permission.CHANGE_CONFIGURATION" />** *在< activity标签中添加:***android:configChanges="orientation"** *将targetSdkVersion改为12以上的,12也可以* 

#### 3.7 AsyncTask异步任务

> [Android的进程，线程模型](https://www.cnblogs.com/ghj1976/archive/2011/04/28/2031586.html)	<=	特别优秀的文章！强烈建议阅读

> 本节给大家带来的是Android给我们提供的一个轻量级的用于处理异步任务的类:AsyncTask，我们一般是 继承AsyncTask，然后在类中实现异步操作，然后将异步执行的进度，反馈给UI主线程~ 好吧，可能有些概念大家不懂，觉得还是有必要讲解下多线程的概念，那就先解释下一些概念性的东西 

##### 1. 相关概念

1. 什么是多线程

   > 答：先要了解这几个名称：应用程序，进程，线程，多线程！！
   >
   > - **应用程序(Application)**：为了完成特定任务，用某种语言编写的一组指令集合(一组静态代码)
   > - **进程(Process)** :**运行中的程序**，系统调度与资源分配的一个**独立单位**，操作系统会为每个进程分配 一段内存空间，程序的依次动态执行，经理代码加载 -> 执行 -> 执行完毕的完整过程！
   > - **线程(Thread)**：比进程更小的执行单元，每个进程可能有多条线程，**线程需要放在一个进程中才能执行！** 线程是由程序负责管理的！！！而进程则是由系统进行调度的！！！
   > - **多线程概念(Multithreading)**：并行地执行多条指令，将CPU的**时间片**按照调度算法，分配给各个线程，实际上是**分时执行**的，只是这个切换的时间很短，用户感觉是同时而已！

2. 同步和异步的概念

3. Android 为什么要引入异步任务

   >  *答：因为Android程序刚启动时，会同时启动一个对应的主线程(Main Thread)，这个主线程主要负责处理 与UI相关的事件！有时我们也把他称作UI线程！而在Android App时我们必须遵守这个单线程模型的规则：* **Android UI操作并不是线程安全的并且这些操作都需要在UI线程中执行！** *假如我们在非UI线程中，比如在主线程中new Thread()另外开辟一个线程，然后直接在里面修改UI控件的值； 此时会抛出下述异常：* **android.view.ViewRoot$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views** *另外，还有一点，如果我们把耗时的操作都放在UI线程中的话，如果UI线程超过5s没有响应用于请求，那么 这个时候会引发ANR(Application Not Responding)异常，就是应用无响应~ 最后还有一点就是：Android 4.0后禁止在UI线程中执行网络操作~不然会报:* **android.os.NetworkOnMainThreadException** 

以上的种种原因都说明了Android引入异步任务的意义，当然实现异步也可以不用到我们本节讲解 的AsyncTask，我们可以自己开辟一个线程，完成相关操作后，通过下述两种方法进行UI更新：

> 1. 前面我们学的Handler，我们在Handler里写好UI更新，然后通过sendMessage()等的方法通知UI 更新，另外别忘了Handler写在主线程和子线程中的区别哦~ 
> 2. 利用Activity.runOnUiThread(Runnable)把更新ui的代码创建在Runnable中,更新UI时，把Runnable 对象传进来即可~

##### 2. AsyncTask全解析

1. 为什么要用AsyncTask？

答:我们可以用上述两种方法来完成我们的异步操作，假如要我们写的异步操作比较多，或者较为繁琐， 难道我们new Thread()然后用上述方法通知UI更新么？程序员都是比较喜欢偷懒的，既然官方给我 们提供了AsyncTask这个封装好的轻量级异步类，为什么不用呢？我们通过几十行的代码就可以完成 我们的异步操作，而且进度可控；相比起Handler，AsyncTask显得更加简单，快捷~当然，这只适合 简单的异步操作，另外，实际异步用的最多的地方就是网络操作，图片加载，数据传输等，AsyncTask 暂时可以满足初学者的需求，比如小应用，但是到了公司真正做项目以后，我们更多的使用第三发的 框架，比如Volley,OkHttp,android-async-http,XUtils等很多，后面进阶教程我们会选1-2个框架进行 学习，当然你可以自己找资料学习学习，但是掌握AsyncTask还是很有必要的！ 

2. AsyncTask的基本结构

 AsyncTask是一个抽象类，一般我们都会定义一个类继承AsyncTask然后重写相关方法~ 官方API:[AsyncTask](https://www.runoob.com/wp-content/uploads/2015/07/39584771.jpg) 

+  **构建AsyncTask子类的参数：** 

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/39584771.jpg) 

+  **相关方法与执行流程：** 

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/27686655.jpg) 

+ 注意事项

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/98978225.jpg) 

##### 3. AsyncTask使用示例

 这里用延时线程来模拟文件下载的过程

下面就贴部分示例代码:

 **定义一个延时操作，用于模拟下载** 

```java
public class DelayOperator {  
    //延时操作,用来模拟下载  
    public void delay()  
    {  
        try {  
            Thread.sleep(1000);  
        }catch (InterruptedException e){  
            e.printStackTrace();;  
        }  
    }  
}
```

 **自定义AsyncTask** 

```java
public class MyAsyncTask extends AsyncTask<Integer,Integer,String>  
{  
    private TextView txt;  
    private ProgressBar pgbar;  
  
    public MyAsyncTask(TextView txt,ProgressBar pgbar)  
    {  
        super();  
        this.txt = txt;  
        this.pgbar = pgbar;  
    }  
  
  
    //该方法不运行在UI线程中,主要用于异步操作,通过调用publishProgress()方法  
    //触发onProgressUpdate对UI进行操作  
    @Override  
    protected String doInBackground(Integer... params) {  
        DelayOperator dop = new DelayOperator();  
        int i = 0;  
        for (i = 10;i <= 100;i+=10)  
        {  
            dop.delay();  
            publishProgress(i);  
        }  
        return  i + params[0].intValue() + "";  
    }  
  
    //该方法运行在UI线程中,可对UI控件进行设置  
    @Override  
    protected void onPreExecute() {  
        txt.setText("开始执行异步线程~");  
    }  
  
  
    //在doBackground方法中,每次调用publishProgress方法都会触发该方法  
    //运行在UI线程中,可对UI控件进行操作  
  
  
    @Override  
    protected void onProgressUpdate(Integer... values) {  
        int value = values[0];  
        pgbar.setProgress(value);  
    }  
}
```

 **MainActivity.java** 

```java
public class MyActivity extends ActionBarActivity {  
  
    private TextView txttitle;  
    private ProgressBar pgbar;  
    private Button btnupdate;  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        txttitle = (TextView)findViewById(R.id.txttitle);  
        pgbar = (ProgressBar)findViewById(R.id.pgbar);  
        btnupdate = (Button)findViewById(R.id.btnupdate);  
        btnupdate.setOnClickListener(new View.OnClickListener() {  
            @Override  
            public void onClick(View v) {  
                MyAsyncTask myTask = new MyAsyncTask(txttitle,pgbar);  
                myTask.execute(1000);  
            }  
        });  
    }  
} 
```

#### 3.8 Gestures(手势)

##### 1. Android中手势交互的执行顺序

1. 手指触碰屏幕时，触发MotionEvent事件！

2. 该事件被OnTouchListener监听，可在它的onTouch()方法中获得该MotionEvent对象！

3. 通过GestureDetector转发MotionEvent对象给OnGestureListener

4. 我们可以通过OnGestureListener获得该对象，然后获取相关信息，以及做相关处理！

我们来看下上述的三个类都是干嘛的: **MotionEvent**: 这个类用于封装手势、触摸笔、轨迹球等等的动作事件。 其内部封装了两个重要的属性X和Y，这两个属性分别用于记录横轴和纵轴的坐标。 **GestureDetector**: 识别各种手势。 **OnGestureListener**: 这是一个手势交互的监听接口，其中提供了多个抽象方法， 并根据GestureDetector的手势识别结果调用相对应的方法。

——上述资料摘自:http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2012/1020/448.html

##### 2. GestureListener详解

GestureListener提供的回调方法

> - 按下（onDown）： 刚刚手指接触到触摸屏的那一刹那，就是触的那一下。
> - 抛掷（onFling）： 手指在触摸屏上迅速移动，并松开的动作。
> - 长按（onLongPress）： 手指按在持续一段时间，并且没有松开。
> - 滚动（onScroll）： 手指在触摸屏上滑动。
> - 按住（onShowPress）： 手指按在触摸屏上，它的时间范围在按下起效，在长按之前。
> - 抬起（onSingleTapUp）：手指离开触摸屏的那一刹那。

实现手势检测的步骤：

> Step 1: 创建GestureDetector对象，创建时需实现GestureListener传入
>
> Step 2: 将Activity或者特定组件上的TouchEvent的事件交给GestureDetector处理即可！ 我们写个简单的代码来验证这个流程，即重写对应的方法：

代码示例：

```java
public class MainActivity extends AppCompatActivity {

    private MyGestureListener mgListener;
    private GestureDetector mDetector;
    private final static String TAG = "MyGesture";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //实例化GestureListener与GestureDetector对象
        mgListener = new MyGestureListener();
        mDetector = new GestureDetector(this, mgListener);

    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return mDetector.onTouchEvent(event);
    }

    //自定义一个GestureListener,这个是View类下的，别写错哦！！！
    private class MyGestureListener implements GestureDetector.OnGestureListener {

        @Override
        public boolean onDown(MotionEvent motionEvent) {
            Log.d(TAG, "onDown:按下");
            return false;
        }

        @Override
        public void onShowPress(MotionEvent motionEvent) {
            Log.d(TAG, "onShowPress:手指按下一段时间,不过还没到长按");
        }

        @Override
        public boolean onSingleTapUp(MotionEvent motionEvent) {
            Log.d(TAG, "onSingleTapUp:手指离开屏幕的一瞬间");
            return false;
        }

        @Override
        public boolean onScroll(MotionEvent motionEvent, MotionEvent motionEvent1, float v, float v1) {
            Log.d(TAG, "onScroll:在触摸屏上滑动");
            return false;
        }

        @Override
        public void onLongPress(MotionEvent motionEvent) {
            Log.d(TAG, "onLongPress:长按并且没有松开");
        }

        @Override
        public boolean onFling(MotionEvent motionEvent, MotionEvent motionEvent1, float v, float v1) {
            Log.d(TAG, "onFling:迅速滑动，并松开");
            return false;
        }
    }

}
```

对应操作截图：

- 1.按下后立即松开:![img](https://www.runoob.com/wp-content/uploads/2015/07/80514897.jpg)
- 2.长按后松开：![img](https://www.runoob.com/wp-content/uploads/2015/07/22745782.jpg)
- 3.轻轻一滑，同时松开：![img](https://www.runoob.com/wp-content/uploads/2015/07/36633841.jpg)
- 4.按住后不放持续做滑动操作：![img](https://www.runoob.com/wp-content/uploads/2015/07/45064641.jpg)

PS:从上述结果来看，我们发现了一个问题： <u>我们实现OnGestureListener需要实现所有的手势，可能我针对的仅仅是滑动，但是你还是要去重载</u>， 这显得很逗逼，是吧，官方肯定会给出解决方法滴，官方另外给我们提供了一个**SimpleOnGestureListener**类 只需把上述的OnGestureListener替换成SimpleOnGestureListener即可！

##### 3. 简单的例子:下滑关闭Activity，上滑启动新的Activity

 这里就用上述的SimpleOnGestureListener来实现吧 

```java 
public class MainActivity extends AppCompatActivity {

    private GestureDetector mDetector;
    private final static int MIN_MOVE = 200;   //最小距离
    private MyGestureListener mgListener;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //实例化SimpleOnGestureListener与GestureDetector对象
        mgListener = new MyGestureListener();
        mDetector = new GestureDetector(this, mgListener);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return mDetector.onTouchEvent(event);
    }

    //自定义一个GestureListener,这个是View类下的，别写错哦！！！
    private class MyGestureListener extends GestureDetector.SimpleOnGestureListener {

        @Override
        public boolean onFling(MotionEvent e1, MotionEvent e2, float v, float v1) {
            if(e1.getY() - e2.getY() > MIN_MOVE){
                startActivity(new Intent(MainActivity.this, MainActivity.class));
                Toast.makeText(MainActivity.this, "通过手势启动Activity", Toast.LENGTH_SHORT).show();
            }else if(e1.getY() - e2.getY()  < MIN_MOVE){
                finish();
                Toast.makeText(MainActivity.this,"通过手势关闭Activity",Toast.LENGTH_SHORT).show();
            }
            return true;
        }
    }

}
```

 **结果分析：** 从上面的对比就可以知道，相比起SimpleOnGestureListener使用SimpleOnGestureListener 显得更加的简单，想重写什么方法就重写什么方法，另外例子比较简单，大家可以自己试试 其他玩法，比如通过手势缩放图片 

##### 4. 手势添加与识别

>  *除了上面讲解的手势检测外，Android还允许我们将手势进行添加，然后提供了相关的识别API； Android中使用GestureLibrary来代表手势库，提供了GestureLibraries工具类来创建手势库* 

 **四个加载手势库的静态方法** 

 ![img](https://www.runoob.com/wp-content/uploads/2015/07/51353345.jpg) 

 获得GestureLibraries对象后，就可以使用该对象提供的下述方法来做相应操作了：

**相关方法：**

- public void **addGesture** (String entryName, Gesture gesture)：添加一个名为entryName的手势
- public Set\<String\> **getGestureEntries** ()：获得手势库中所有手势的名称
- public ArrayList\<Gesture\> **getGestures** (String entryName)：获得entryName名称对应的全部手势
- public ArrayList<Prediction\> **recognize** (Gesture gesture): 从当前手势库中识别与gesture匹配的全部手势
- public void **removeEntry** (String entryName)：删除手势库中entryName名称对应的手势
- public void **removeGesture** (String entryName, Gesture gesture)：删除手势库中entryName和gesture都匹配的手势
- public abstract boolean **save** ()：向手势库中添加手势或从中删除手势后调用该方法保存手势库

**GestureOverlayView手势编辑组件：**

Android为GestureOverlayView提供了三种监听器接口，如下，一般常用的是:**OnGesturePerformedListener**;用于手势完成时提供响应！

![img](https://www.runoob.com/wp-content/uploads/2015/07/55343440.jpg)

##### 5. 手势添加示例

##### 6. 手势识别示例

### 第四章 Android的四大组件

本节开始讲解Android的四大组件之一的Activity(活动)，先来看下官方对于Activity的介绍： PS:官网文档：[Activity](http://androiddoc.qiniudn.com/guide/components/activities.html) 

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/48284031.jpg) 

##### 1. Activity的概念与Activity的生命周期图

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/18364230.jpg) 

**注意事项**

> 1.**onPause()和onStop()被调用的前提是**： 打开了一个新的Activity！而前者是旧Activity还可见的状态；后者是旧Activity已经不可见！
> 2.另外，亲测：AlertDialog和PopWindow是不会触发上述两个回调方法的 

##### 2. Activity/ActionBarActivity/AppCompatActivity的区别

>  *在开始讲解创建Activity之前要说下这三个的一个区别： Activity就不用说啦，后面这两个都是为了低版本兼容而提出的提出来的，他们都在v7包下， ActionBarActivity已被废弃，从名字就知道，ActionBar~，而在5.0后，被Google弃用了，现在用 ToolBar...而我们现在在Android Studio创建一个Activity默认继承的会是：AppCompatActivity! 当然你也可以只写Activity，不过AppCompatActivity给我们提供了一些新的东西而已！ 两个选一个，Just you like* 

##### 3. Activity的创建流程

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/48768883.jpg) 

注意：

>  *上面也说过，可以继承Activity和AppCompatActivity，只不过后者提供了一些新的东西而已！ 另外，切记，Android中的四大组件，只要你定义了，无论你用没用，都要在AndroidManifest.xml对 这个组件进行声明，不然运行时程序会直接退出，报ClassNotFindException...* 

##### 4. onCreate()一个参数和两个参数的区别

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/18677320.jpg) 

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/28609433.jpg) 

 这就是5.0给我们提供的新的方法，要用它，先要在配置文件中为我们的Activity设置一个属性 

```xml
android:persistableMode="persistAcrossReboots"
```

 然后我们的Activity就拥有了持久化的能力了，一般我们会搭配另外两个方法来使用 

```java
public void onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState)
public void onRestoreInstanceState(Bundle savedInstanceState, PersistableBundle persistentState)
```

 前一个方法会在下述情形中被调用 

> 1. 点击home键回到主页或长按后选择运行其他程序
> 2. 按下电源键关闭屏幕
> 3. 启动新的Activity
> 4. 横竖屏切换时，肯定会执行，因为横竖屏切换的时候会先销毁Act，然后再重新创建 重要原则：当系统"未经你许可"时销毁了你的activity，则onSaveInstanceState会被系统调用， 这是系统的责任，因为它必须要提供一个机会让你保存你的数据（你可以保存也可以不保存）

而后一个方法，和onCreate同样可以从取出前者保存的数据： 一般是在onStart()和onResume()之间执行！ 之所以有两个可以获取到保存数据的方法，是为了避免Act跳转而没有关闭， 然后不走onCreate()方法，而你又想取出保存数据~

**说回来：** 说回这个Activity拥有了持久化的能力，增加的这个PersistableBundle参数令这些方法 拥有了系统**关机后重启**的数据恢复能力！！而且不影响我们其他的序列化操作，卧槽， 具体怎么实现的，暂时还不了解，可能是另外弄了个文件保存吧~！后面知道原理的话会告知下大家！ 另外，API版本需要>=21，就是要5.0以上的版本才有效

##### 4. 启动一个Activity的几种方式

> *在Android中我们可以通过下面两种方式来启动一个新的Activity,注意这里是怎么启动，而非 启动模式！！分为显示启动和隐式启动* 

1.   **显式启动：通过包名来启动，写法如下** 

   **①最常见的**

   ```java
   startActivity(new Intent(当前Act.this,要启动的Act.class));
   ```

    **②通过Intent的ComponentName**

   ```java
   ComponentName cn = new ComponentName("当前Act的全限定类名","启动Act的全限定类名") ;
   Intent intent = new Intent() ;
   intent.setComponent(cn) ;
   startActivity(intent) ;
   ```

    ③**初始化Intent时指定包名**

   ```java
   Intent intent = new Intent("android.intent.action.MAIN");
   intent.setClassName("当前Act的全限定类名","启动Act的全限定类名");
   startActivity(intent);
   ```

2.  **隐式启动：通过Intent-filter的Action,Category或data来实现 这个是通过Intent的** intent-filter**来实现的，这个Intent那章会详细讲解！ 这里知道个大概就可以了

    ![img](https://www.runoob.com/wp-content/uploads/2015/08/291262381.jpg) 

3.  **另外还有一个直接通过包名启动apk的** 

   ```java
   Intent intent = getPackageManager().getLaunchIntentForPackage
   ("apk第一个启动的Activity的全限定类名") ;
   if(intent != null) startActivity(intent) ;
   ```

##### 5. 横竖屏切换与状态保存的问题

>  *前面也也说到了App横竖屏切换的时候会销毁当前的Activity然后重新创建一个，你可以自行在生命周期 的每个方法里都添加打印Log的语句，来进行判断，又或者设一个按钮一个TextView点击按钮后，修改TextView 文本，然后横竖屏切换，会神奇的发现TextView文本变回之前的内容了！ 横竖屏切换时Act走下述生命周期：*
> **onPause-> onStop-> onDestory-> onCreate->onStart->onResume** 

1.  先说下如何**禁止屏幕横竖屏自动切换**吧，很简单，在AndroidManifest.xml中为Act添加一个属性： **android:screenOrientation**， 有下述可选值 

   - **unspecified**:默认值 由系统来判断显示方向.判定的策略是和设备相关的，所以不同的设备会有不同的显示方向。
   - **landscape**:横屏显示（宽比高要长）
   - **portrait**:竖屏显示(高比宽要长)
   - **user**:用户当前首选的方向
   - **behind**:和该Activity下面的那个Activity的方向一致(在Activity堆栈中的)
   - **sensor**:有物理的感应器来决定。如果用户旋转设备这屏幕会横竖屏切换。
   - **nosensor**:忽略物理感应器，这样就不会随着用户旋转设备而更改了（"unspecified"设置除外）

2. 横屏时想加载不同的布局

   1）准备两套不同的布局，Android会自己根据横竖屏加载不同布局： 创建两个布局文件夹：**layout-land**横屏,**layout-port**竖屏 然后把这两套布局文件丢这两文件夹里，文件名一样，Android就会自行判断，然后加载相应布局了！

   2 )自己在代码中进行判断，自己想加载什么就加载什么：

   我们一般是在onCreate()方法中加载布局文件的，我们可以在这里对横竖屏的状态做下判断，关键代码如下

   ```java
   if (this.getResources().getConfiguration().orientation == Configuration.ORIENTATION_LANDSCAPE){  
        setContentView(R.layout.横屏);
   }  
   
   else if (this.getResources().getConfiguration().orientation ==Configuration.ORIENTATION_PORTRAIT) {  
       setContentView(R.layout.竖屏);
   }
   ```

3.  **如何让模拟器横竖屏切换**

    如果你的模拟器是GM的话。直接按模拟器上的切换按钮即可，原生模拟器可按ctrl + f11/f12切换！ 

4.  **状态保存问题** 

    这个上面也说过了，通过一个Bundle savedInstanceState参数即可完成！ 三个核心方法:

   ```java
   onCreate(Bundle savedInstanceState);
   onSaveInstanceState(Bundle outState);
   onRestoreInstanceState(Bundle savedInstanceState);
   ```

    只重写onSaveInstanceState()方法，往这个bundle中写入数据，比如：

   ```java
   outState.putInt("num",1);
   ```

   这样，然后你在onCreate或者onRestoreInstanceState中就可以拿出里面存储的数据，不过拿之前要判断下是否为null哦 

   ```java
   savedInstanceState.getInt("num");
   ```

##### 6. 系统给我们提供的常见的Activity

 好的，最后给大家附上一些系统给我们提供的一些常见的Activtiy吧 

```java
//1.拨打电话
// 给移动客服10086拨打电话
Uri uri = Uri.parse("tel:10086");
Intent intent = new Intent(Intent.ACTION_DIAL, uri);
startActivity(intent);

//2.发送短信
// 给10086发送内容为“Hello”的短信
Uri uri = Uri.parse("smsto:10086");
Intent intent = new Intent(Intent.ACTION_SENDTO, uri);
intent.putExtra("sms_body", "Hello");
startActivity(intent);

//3.发送彩信（相当于发送带附件的短信）
Intent intent = new Intent(Intent.ACTION_SEND);
intent.putExtra("sms_body", "Hello");
Uri uri = Uri.parse("content://media/external/images/media/23");
intent.putExtra(Intent.EXTRA_STREAM, uri);
intent.setType("image/png");
startActivity(intent);

//4.打开浏览器:
// 打开Google主页
Uri uri = Uri.parse("http://www.baidu.com");
Intent intent  = new Intent(Intent.ACTION_VIEW, uri);
startActivity(intent);

//5.发送电子邮件:(阉割了Google服务的没戏!!!!)
// 给someone@domain.com发邮件
Uri uri = Uri.parse("mailto:someone@domain.com");
Intent intent = new Intent(Intent.ACTION_SENDTO, uri);
startActivity(intent);
// 给someone@domain.com发邮件发送内容为“Hello”的邮件
Intent intent = new Intent(Intent.ACTION_SEND);
intent.putExtra(Intent.EXTRA_EMAIL, "someone@domain.com");
intent.putExtra(Intent.EXTRA_SUBJECT, "Subject");
intent.putExtra(Intent.EXTRA_TEXT, "Hello");
intent.setType("text/plain");
startActivity(intent);
// 给多人发邮件
Intent intent=new Intent(Intent.ACTION_SEND);
String[] tos = {"1@abc.com", "2@abc.com"}; // 收件人
String[] ccs = {"3@abc.com", "4@abc.com"}; // 抄送
String[] bccs = {"5@abc.com", "6@abc.com"}; // 密送
intent.putExtra(Intent.EXTRA_EMAIL, tos);
intent.putExtra(Intent.EXTRA_CC, ccs);
intent.putExtra(Intent.EXTRA_BCC, bccs);
intent.putExtra(Intent.EXTRA_SUBJECT, "Subject");
intent.putExtra(Intent.EXTRA_TEXT, "Hello");
intent.setType("message/rfc822");
startActivity(intent);

//6.显示地图:
// 打开Google地图中国北京位置（北纬39.9，东经116.3）
Uri uri = Uri.parse("geo:39.9,116.3");
Intent intent = new Intent(Intent.ACTION_VIEW, uri);
startActivity(intent);

//7.路径规划
// 路径规划：从北京某地（北纬39.9，东经116.3）到上海某地（北纬31.2，东经121.4）
Uri uri = Uri.parse("http://maps.google.com/maps?f=d&saddr=39.9 116.3&daddr=31.2 121.4");
Intent intent = new Intent(Intent.ACTION_VIEW, uri);
startActivity(intent);

//8.多媒体播放:
Intent intent = new Intent(Intent.ACTION_VIEW);
Uri uri = Uri.parse("file:///sdcard/foo.mp3");
intent.setDataAndType(uri, "audio/mp3");
startActivity(intent);

//获取SD卡下所有音频文件,然后播放第一首=-= 
Uri uri = Uri.withAppendedPath(MediaStore.Audio.Media.INTERNAL_CONTENT_URI, "1");
Intent intent = new Intent(Intent.ACTION_VIEW, uri);
startActivity(intent);

//9.打开摄像头拍照:
// 打开拍照程序
Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE); 
startActivityForResult(intent, 0);
// 取出照片数据
Bundle extras = intent.getExtras(); 
Bitmap bitmap = (Bitmap) extras.get("data");

//另一种:
//调用系统相机应用程序，并存储拍下来的照片
Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE); 
time = Calendar.getInstance().getTimeInMillis();
intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(new File(Environment
.getExternalStorageDirectory().getAbsolutePath()+"/tucue", time + ".jpg")));
startActivityForResult(intent, ACTIVITY_GET_CAMERA_IMAGE);

//10.获取并剪切图片
// 获取并剪切图片
Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
intent.setType("image/*");
intent.putExtra("crop", "true"); // 开启剪切
intent.putExtra("aspectX", 1); // 剪切的宽高比为1：2
intent.putExtra("aspectY", 2);
intent.putExtra("outputX", 20); // 保存图片的宽和高
intent.putExtra("outputY", 40); 
intent.putExtra("output", Uri.fromFile(new File("/mnt/sdcard/temp"))); // 保存路径
intent.putExtra("outputFormat", "JPEG");// 返回格式
startActivityForResult(intent, 0);
// 剪切特定图片
Intent intent = new Intent("com.android.camera.action.CROP"); 
intent.setClassName("com.android.camera", "com.android.camera.CropImage"); 
intent.setData(Uri.fromFile(new File("/mnt/sdcard/temp"))); 
intent.putExtra("outputX", 1); // 剪切的宽高比为1：2
intent.putExtra("outputY", 2);
intent.putExtra("aspectX", 20); // 保存图片的宽和高
intent.putExtra("aspectY", 40);
intent.putExtra("scale", true);
intent.putExtra("noFaceDetection", true); 
intent.putExtra("output", Uri.parse("file:///mnt/sdcard/temp")); 
startActivityForResult(intent, 0);

//11.打开Google Market 
// 打开Google Market直接进入该程序的详细页面
Uri uri = Uri.parse("market://details?id=" + "com.demo.app");
Intent intent = new Intent(Intent.ACTION_VIEW, uri);
startActivity(intent);

//12.进入手机设置界面:
// 进入无线网络设置界面（其它可以举一反三）  
Intent intent = new Intent(android.provider.Settings.ACTION_WIRELESS_SETTINGS);  
startActivityForResult(intent, 0);

//13.安装apk:
Uri installUri = Uri.fromParts("package", "xxx", null);   
returnIt = new Intent(Intent.ACTION_PACKAGE_ADDED, installUri);

//14.卸载apk:
Uri uri = Uri.fromParts("package", strPackageName, null);      
Intent it = new Intent(Intent.ACTION_DELETE, uri);      
startActivity(it); 

//15.发送附件:
Intent it = new Intent(Intent.ACTION_SEND);      
it.putExtra(Intent.EXTRA_SUBJECT, "The email subject text");      
it.putExtra(Intent.EXTRA_STREAM, "file:///sdcard/eoe.mp3");      
sendIntent.setType("audio/mp3");      
startActivity(Intent.createChooser(it, "Choose Email Client"));

//16.进入联系人页面:
Intent intent = new Intent();
intent.setAction(Intent.ACTION_VIEW);
intent.setData(People.CONTENT_URI);
startActivity(intent);

//17.查看指定联系人:
Uri personUri = ContentUris.withAppendedId(People.CONTENT_URI, info.id);//info.id联系人ID
Intent intent = new Intent();
intent.setAction(Intent.ACTION_VIEW);
intent.setData(personUri);
startActivity(intent);
```

#### 4.1.2 Activity初窥门径

> 上一节中我们对Activity一些基本的概念进行了了解，什么是Activity，Activity的生命周期，如何去启动一个Activity等，本节我们继续来学习Activity，前面也讲了一个App一般都是又多个Activity构成的，这就涉及到了多个Activity间数据传递的问题了，那么本节继续学习Activity的使用！另外关于传递集合，对象，数组，Bitmap的我们会在Intent那里进行讲解，这里只介绍如何传递基本数据 

##### 1. Activity间的数据传递

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/7185831.jpg) 

##### 2. 多个Activity间的交互(后一个传回给前一个)

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/67124491.jpg) 

这里贴部分代码：

```java
// MyActivity
public class MyActivity extends ActionBarActivity {

    private Button btnchoose;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my);
        btnchoose = (Button)findViewById(R.id.btnchoose);
        btnchoose.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent it = new Intent(MyActivity.this,MyActivity2.class);
                startActivityForResult(it,0x123);
            }
        });
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if(requestCode == 0x123 && resultCode == 0x123)
        {
            Bundle bd = data.getExtras();
            int imgid = bd.getInt("imgid");
            //获取布局文件中的ImageView组件
            ImageView img = (ImageView)findViewById(R.id.imgicon);
            img.setImageResource(imgid);
        }
    }
}


// MyActivity2 其中gd是GridView
gd.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Intent it = getIntent();
                Bundle bd = new Bundle();
                bd.putInt("imgid",imgs[position]);
                it.putExtras(bd);
                setResult(0x123,it);
                finish();
            }
        });
```

##### 3. 知晓当前是哪个Activity

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/19579941.jpg) 

##### 4. 随时关闭所有Activity

> *有时我们可能会打开了很多个Activity，突然来个这样的需求，在某个页面可以关掉 所有的Activity并退出程序！好吧，下面提供一个关闭所有Activity的方法， 就是用一个list集合来存储所有Activity!*

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/59443692.jpg) 

**这里我觉得应该用Set，而不是用List，之前看过一篇文章，如果存储的元素不应该重复，那么最好直接用Set，而不是用List**

##### 5. 完全退出App的方法

 上面说的是关闭所有Activity的，但是有些时候我们可能想杀死整个App，连后台任务都杀死杀得一干二净的话，可以使用搭配着下述代码使用

```java
/** 
 * 退出应用程序 
 */  
public void AppExit(Context context) {  
    try {  
        ActivityCollector.finishAll();  
        ActivityManager activityMgr = (ActivityManager) context  
                .getSystemService(Context.ACTIVITY_SERVICE);  
        activityMgr.killBackgroundProcesses(context.getPackageName());  
        System.exit(0);  
    } catch (Exception ignored) {}  
}  
```

##### 6. 双击退出程序的两种方法

1. 定义一个变量，来标识是否退出

   ```java
   // 定义一个变量，来标识是否退出
   private static boolean isExit = false;
   Handler mHandler = new Handler() {
       @Override
       public void handleMessage(Message msg) {
           super.handleMessage(msg);
           isExit = false;
       }
   };
   
   public boolean onKeyDown(int keyCode, KeyEvent event) {
       if (keyCode == KeyEvent.KEYCODE_BACK) {
           if (!isExit) {
               isExit = true;
               Toast.makeText(getApplicationContext(), "再按一次退出程序",
                       Toast.LENGTH_SHORT).show();
               // 利用handler延迟发送更改状态信息
               mHandler.sendEmptyMessageDelayed(0, 2000);
           } else {
               exit(this);
           }
           return false;
       }
   return super.onKeyDown(keyCode, event);}
   ```

2. 保存点击时间

   ```java
   //保存点击的时间
   private long exitTime = 0;
   public boolean onKeyDown(int keyCode, KeyEvent event) {
       if (keyCode == KeyEvent.KEYCODE_BACK) {
           if ((System.currentTimeMillis() - exitTime) > 2000) {
               Toast.makeText(getApplicationContext(), "再按一次退出程序",
                       Toast.LENGTH_SHORT).show();
               exitTime = System.currentTimeMillis();
           } else {
                           exit();
                         }
           return false;
       }
           return super.onKeyDown(keyCode, event);
   }
   ```

#### 7. 为Activity设置过场动画

> *所谓的过场动画就是切换到另外的Activity时加上一些切换动画，比如淡入淡出，放大缩小，左右互推等！ 当然，我们并不在这里详细讲解动画，后面有专门的章节来讲解这个，这里只教大家如何去加载动画，另外 给大家提供了一些比较常用的过渡动画，只要将相关动画文件添加到res/anim目录下，然后下述方法二选一 就可以实现Activity的切换动画了* 

1. 方法一

    ![img](https://www.runoob.com/wp-content/uploads/2015/08/16878455.jpg) 

2. 方法二

    通过style进行配置，这个是全局的哦，就是所有的Activity都会加载这个动画 
   
   **实现代码如下：**
   
   **①在style.xml中自定义style：**
   
   ```xml
   <!-- 默认Activity跳转动画 -->
   <style name="default_animation" mce_bogus="1" parent="@android:style/Animation.Activity">
       <item name="android:activityOpenEnterAnimation">@anim/default_anim_in</item>
       <item name="android:activityOpenExitAnimation">@anim/anim_stay</item>
       <item name="android:activityCloseEnterAnimation">@anim/anim_stay</item>
       <item name="android:activityCloseExitAnimation">@anim/default_anim_out</item>
   </style>
   ```
   
   **解释：**
   
   4个item分别代表:
   
   - Activity A跳转到Activity B时Activity B进入动画;
   - Activity A跳转到Activity B时Activity A退出动画;
   - Activity B返回Activity A时Activity A的进入动画
   - Activity B返回Activity A时Activity B的退出动画
   
    **②然后修改下AppTheme:** 
   
   ```xml
   <style name="AppTheme" mce_bogus="1" parent="@android:style/Theme.Light">
           <item name="android:windowAnimationStyle">@style/default_animation</item>
           <item name="android:windowNoTitle">true</item>
   </style>
   ```
   
    **③最后在appliction设置下：** 
   
   ```xml
   <application
      android:icon="@drawable/logo"
      android:label="@string/app_name"
      android:theme="@style/AppTheme" >
   ```

3. 其他

   ## 4.1.2 Activity初窥门径

   ### *分类* [Android 基础入门教程](https://www.runoob.com/w3cnote_genre/android)

   ## 本节引言：

   上一节中我们对Activity一些基本的概念进行了了解，什么是Activity，Activity的生命周期，如何去启动一个Activity等，本节我们继续来学习Activity，前面也讲了一个App一般都是又多个Activity构成的，这就涉及到了多个Activity间数据传递的问题了，那么本节继续学习Activity的使用！另外关于传递集合，对象，数组，Bitmap的我们会在Intent那里进行讲解，这里只介绍如何传递基本数据！

   ------

   ## 1.Activity间的数据传递：

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/7185831.jpg)

   **代码示例：**

   **效果图：**

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/65736101.jpg)

   **代码下载：**[ActivityTest1.zip](https://www.runoob.com/try/download/ActivityTest1.zip)

   ------

   ## 2.多个Activity间的交互(后一个传回给前一个)

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/67124491.jpg)

   **代码示例：**

   **效果图：**

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/41632576.jpg)

   **代码下载：**[ActivityTest2.zip](https://www.runoob.com/try/download/ActivityTest2.zip)

   ------

   ## 3.知晓当前是哪个Activity

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/19579941.jpg)

   ------

   ## 4.随时关闭所有Activity

   > 有时我们可能会打开了很多个Activity，突然来个这样的需求，在某个页面可以关掉 所有的Activity并退出程序！好吧，下面提供一个关闭所有Activity的方法， 就是用一个list集合来存储所有Activity!

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/59443692.jpg)

   **具体代码如下：**

   ```
   public class ActivityCollector {  
       public static LinkedList<Activity> activities = new LinkedList<Activity>();  
       public static void addActivity(Activity activity)  
       {  
           activities.add(activity);  
       }  
       public static void removeActivity(Activity activity)  
       {  
           activities.remove(activity);  
       }  
       public static void finishAll()  
       {  
           for(Activity activity:activities)  
           {  
               if(!activity.isFinishing())  
               {  
                   activity.finish();  
               }  
           }  
       }  
   }  
   ```

   ------

   ## 5.完全退出App的方法

   上面说的是关闭所有Activity的，但是有些时候我们可能想杀死整个App，连后台任务都杀死 杀得一干二净的话，可以使用搭配着下述代码使用：

   **实现代码：**

   ```
   /** 
    * 退出应用程序 
    */  
   public void AppExit(Context context) {  
       try {  
           ActivityCollector.finishAll();  
           ActivityManager activityMgr = (ActivityManager) context  
                   .getSystemService(Context.ACTIVITY_SERVICE);  
           activityMgr.killBackgroundProcesses(context.getPackageName());  
           System.exit(0);  
       } catch (Exception ignored) {}  
   }  
   ```

   ------

   ## 6.双击退出程序的两种方法：

   ### 1）定义一个变量，来标识是否退出

   ```
   // 定义一个变量，来标识是否退出
   private static boolean isExit = false;
   Handler mHandler = new Handler() {
       @Override
       public void handleMessage(Message msg) {
           super.handleMessage(msg);
           isExit = false;
       }
   };
   
   public boolean onKeyDown(int keyCode, KeyEvent event) {
       if (keyCode == KeyEvent.KEYCODE_BACK) {
           if (!isExit) {
               isExit = true;
               Toast.makeText(getApplicationContext(), "再按一次退出程序",
                       Toast.LENGTH_SHORT).show();
               // 利用handler延迟发送更改状态信息
               mHandler.sendEmptyMessageDelayed(0, 2000);
           } else {
               exit(this);
           }
           return false;
       }
   return super.onKeyDown(keyCode, event);}
   ```

   ------

   ### 2）保存点击时间：

   ```
   //保存点击的时间
   private long exitTime = 0;
   public boolean onKeyDown(int keyCode, KeyEvent event) {
       if (keyCode == KeyEvent.KEYCODE_BACK) {
           if ((System.currentTimeMillis() - exitTime) > 2000) {
               Toast.makeText(getApplicationContext(), "再按一次退出程序",
                       Toast.LENGTH_SHORT).show();
               exitTime = System.currentTimeMillis();
           } else {
                           exit();
                         }
           return false;
       }
           return super.onKeyDown(keyCode, event);
   }
   ```

   ------

   ## 7.为Activity设置过场动画

   > 所谓的过场动画就是切换到另外的Activity时加上一些切换动画，比如淡入淡出，放大缩小，左右互推等！ 当然，我们并不在这里详细讲解动画，后面有专门的章节来讲解这个，这里只教大家如何去加载动画，另外 给大家提供了一些比较常用的过渡动画，只要将相关动画文件添加到res/anim目录下，然后下述方法二选一 就可以实现Activity的切换动画了！

   ### 1）方法一：

   ![img](https://www.runoob.com/wp-content/uploads/2015/08/16878455.jpg)

   ### 2）方法二：

   通过style进行配置，这个是全局的哦，就是所有的Activity都会加载这个动画！

   **实现代码如下：**

   **①在style.xml中自定义style：**

   ```
   <!-- 默认Activity跳转动画 -->
   <style name="default_animation" mce_bogus="1" parent="@android:style/Animation.Activity">
       <item name="android:activityOpenEnterAnimation">@anim/default_anim_in</item>
       <item name="android:activityOpenExitAnimation">@anim/anim_stay</item>
       <item name="android:activityCloseEnterAnimation">@anim/anim_stay</item>
       <item name="android:activityCloseExitAnimation">@anim/default_anim_out</item>
   </style>
   ```

   **解释：**

   4个item分别代表:

   - Activity A跳转到Activity B时Activity B进入动画;
   - Activity A跳转到Activity B时Activity A退出动画;
   - Activity B返回Activity A时Activity A的进入动画
   - Activity B返回Activity A时ActivityB的退出动画

   **②然后修改下AppTheme:**

   ```
   <style name="AppTheme" mce_bogus="1" parent="@android:style/Theme.Light">
           <item name="android:windowAnimationStyle">@style/default_animation</item>
           <item name="android:windowNoTitle">true</item>
   </style>
   ```

   **③最后在appliction设置下：**

   ```
   <application
      android:icon="@drawable/logo"
      android:label="@string/app_name"
      android:theme="@style/AppTheme" >
   ```

   好的，动画特效就这样duang一声设置好了~

3. 其他

   好的，除了上面两种方法以外，还可以使用**TransitionManager**来实现，但是需求版本是API 19以上的， 另外还有一种**addOnPreDrawListener**的转换动画，这个用起来还是有点麻烦的，可能不是适合初学者 这里也不讲，最后提供下一些常用的动画效果打包，选择需要的特效加入工程即可！ [Activity常用过渡动画.zip](https://www.runoob.com/wp-content/uploads/2015/08/Activity常用过渡动画.zip)

##### 8. Bundle传递数据的限制

> *在使用Bundle传递数据时，要注意，**Bundle的大小是有限制的 < 0.5MB**，如果大于这个值 是会报TransactionTooLargeException异常的* 

##### 9. 使用命令行查看当前所有Activity的命令

##### 10. 设置Activity全屏的方法

1. 代码隐藏ActionBar

    在Activity的onCreate方法中调用getActionBar.hide();即可 

2. 通过requestWindowFeature设置

    requestWindowFeature(Window.FEATURE_NO_TITLE); 该代码需要在setContentView ()之前调用，不然会报错 

   >  **注：** *把 requestWindowFeature(Window.FEATURE_NO_TITLE);放在super.onCreate(savedInstanceState);前面就可以隐藏ActionBar而不报错。* 

3. 通过AndroidManifest.xml的theme
4.  在需要全屏的Activity的标签内设置 theme = @android:style/Theme.NoTitleBar.FullScreen

##### 11. onWindowFocusChanged方法妙用

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/17084157.jpg) 

 就是，当Activity得到或者失去焦点的时候，就会回调该方法！**如果我们想监控Activity是否加载完毕，就可以用到这个方法了**~ 想深入了解的可移步到这篇文章： [onWindowFocusChanged触发简介](http://blog.csdn.net/yueqinglkong/article/details/44981449) 

##### 12. 定义对话框风格的Activity

>  *在某些情况下，我们可能需要将Activity设置成对话框风格的，Activity一般是占满全屏的， 而Dialog则是占据部分屏幕的！实现起来也很简单！* 

 直接设置下Activity的theme

```xml
android:theme="@android:style/Theme.Dialog"
```

 这样就可以了，当然你可以再设置下标题，小图标 

```java 
//设置左上角小图标
requestWindowFeature(Window.FEATURE_LEFT_ICON);
setContentView(R.layout.main);
getWindow().setFeatureDrawableResource(Window.FEATURE_LEFT_ICON, android.R.drawable.ic_lion_icon);
//设置文字:
setTitle(R.string.actdialog_title);  //XML代码中设置:android:label="@string/activity_dialog"
```

#### 4.1.3 Activity登堂入室

##### 1. Activity，Window与View的关系

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/93497523.jpg) 

**流程解析：** Activity调用startActivity后最后会调用attach方法，然后在PolicyManager实现一个Ipolicy接口，接着实现一个Policy对象，接着调用makenewwindow(Context)方法，该方法会返回一个PhoneWindow对象，而PhoneWindow 是Window的子类，在这个PhoneWindow中有一个DecorView的内部类，是所有应用窗口的根View，即View的老大， 直接控制Activity是否显示(引用老司机原话..)，好吧，接着里面有一个LinearLayout，里面又有两个FrameLayout他们分别拿来装ActionBar和CustomView，而我们setContentView()加载的布局就放到这个CustomView中！

**总结下这三者的关系：** 打个牵强的比喻： 我们可以把这三个类分别堪称：画家，画布，画笔画出的东西； 画家通过画笔( **LayoutInflater.infalte**)画出图案，再绘制在画布(**addView**)上！ 最后显示出来(**setContentView**)

##### 2. Activity,Task和Back Stack的一些概念

接着我们来了解Android中Activity的管理机制，这就涉及到了两个名词：Task和Back Stack了！ 

**概念解析：**

我们的APP一般都是由多个Activity构成的，而在Android中给我们提供了一个**Task(任务)**的概念， 就是将多个相关的Activity收集起来，然后进行Activity的跳转与返回！当然，这个Task只是一个 frameworker层的概念，而在Android中实现了Task的数据结构就是**Back Stack（回退堆栈）**！ 相信大家对于栈这种数据结构并不陌生，Java中也有个Stack的集合类！栈具有如下特点：

>  **后进先出(LIFO)，常用操作入栈(push)，出栈(pop)，处于最顶部的叫栈顶，最底部叫栈底** 

 而Android中的Stack Stack也具有上述特点，他是这样来管理Activity的： 

> *当切换到新的Activity，那么该Activity会被压入栈中，成为栈顶！ 而当用户点击Back键，栈顶的Activity出栈，紧随其后的Activity来到栈顶！* 

官方流程图：

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/93537362.jpg) 

 **流程解析：** 

应用程序中存在A1,A2,A3三个activity，当用户在Launcher或Home Screen点击应用程序图标时， 启动主A1，接着A1开启A2，A2开启A3，这时栈中有三个Activity，并且这三个Activity默认在 同一个任务（Task）中，当用户按返回时，弹出A3，栈中只剩A1和A2，再按返回键， 弹出A2，栈中只剩A1，再继续按返回键，弹出A1，任务被移除，即程序退出！

接着在官方文档中又看到了另外两个图，处于好奇，我又看了下解释，然后跟群里的人讨论了下：

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/76132682.jpg) 

 然后还有这段解释： 

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/94874923.jpg) 

 **然后总结下了结论：** 

> Task是Activity的集合，是一个概念，实际使用的Back Stack来存储Activity，可以有多个Task，但是 同一时刻只有一个栈在最前面，其他的都在后台！那栈是如何产生的呢？
>
> 答：当我们通过主屏幕，点击图标打开一个新的App，此时会创建一个新的Task！举个例子：
> 我们通过点击通信录APP的图标打开APP，这个时候会新建一个栈1，然后开始把新产生的Activity添加进来，可能我们在通讯录的APP中打开了短信APP的页面，但是此时不会新建一个栈，而是继续添加到栈1中，这是 Android推崇一种用户体验方式，即不同应用程序之间的切换能使用户感觉就像是同一个应用程序， 很连贯的用户体验，官方称其为seamless (无缝衔接）！ ——————这个时候假如我们点击Home键，回到主屏幕，此时栈1进入后台，我们可能有下述两种操作：
> 1）点击菜单键(正方形那个按钮)，点击打开刚刚的程序，然后栈1又回到前台了！ 又或者我们点击主屏幕上通信录的图标，打开APP，此时也不会创建新的栈，栈1回到前台！
> 2）如果此时我们点击另一个图标打开一个新的APP，那么此时则会创建一个新的栈2，栈2就会到前台， 而栈1继续呆在后台；
> 3) 后面也是这样...以此类推！

##### 3. Task的管理

1. 文档翻译

 继续走文档，从文档中的ManagingTasks开始，翻译如下： 

>  *如上面所述，Android会将新成功启动的Activity添加到同一个Task中并且按照以"先进先出"方式管理多个Task 和Back Stack，用户就无需去担心Activites如何与Task任务进行交互又或者它们是如何存在于Back Stack中！ 或许，你想改变这种正常的管理方式。比如，你希望你的某个Activity能够在一个新的Task中进行管理； 或者你只想对某个Activity进行实例化，又或者你想在用户离开任务时清理Task中除了根Activity所有Activities。你可以做这些事或者更多，只需要通过修改AndroidManifest.xml中 < activity >的相关属性值或者在代码中通过传递特殊标识的Intent给startActivity( )就可以轻松的实现 对Actvitiy的管理了。* 

\<activity\>中我们可以使用的属性如下：

> - **taskAffinity**
> - **launchMode**
> - **allowTaskReparenting**
> - **clearTaskOnLaunch**
> - **alwaysRetainTaskState**
> - **finishOnTaskLaunch**

主要用的Intent标记有

> - **FLAG_ACTIVITY_NEW_TASK**
> - **FLAG_ACTIVITY_CLEAR_TOP**
> - **FLAG_ACTIVITY_SINGLE_TOP**

2. taskAffinity和allowTaskReparenting

 默认情况下，一个应用程序中的**所有activity都有一个Affinity**，这让它们属于同一个Task。 你可以理解为是否处于同一个Task的标志，然而，每个Activity可以通过 < activity>中的taskAffinity属性设置单独的Affinity。 不同应用程序中的Activity可以共享同一个Affinity，同一个应用程序中的不同Activity 也可以设置成不同的Affinity。 Affinity属性在2种情况下起作用： 

> 1）当启动 activity的Intent对象包含**FLAG_ACTIVITY_NEW_TASK**标记： 当传递给startActivity()的Intent对象包含 FLAG_ACTIVITY_NEW_TASK标记时，系统会为需要启动的Activity寻找与当前Activity不同Task。如果要启动的 Activity的Affinity属性与当前所有的Task的Affinity属性都不相同，系统会新建一个带那个Affinity属性的Task，并将要启动的Activity压到新建的Task栈中；否则将Activity压入那个Affinity属性相同的栈中。
>
> 2）**allowTaskReparenting**属性设置为true 如果一个activity的allowTaskReparenting属性为true， 那么它可以从一个Task（Task1）移到另外一个有相同Affinity的Task（Task2）中（Task2带到前台时）。 如果一个.apk文件从用户角度来看包含了多个"应用程序"，你可能需要对那些 Activity赋不同的Affinity值。

3. launchMode:

 四个可选值，启动模式我们研究的核心，下面再详细讲! 他们分别是：**standard**(默认)，**singleTop**，**singleTask**，**singleInstance** 

4. 清空栈

> 当用户长时间离开Task（当前task被转移到后台）时，系统会清除task中栈底Activity外的所有Activity 。这样，当用户返回到Task时，只留下那个task最初始的Activity了。我们可以通过修改下面这些属性来 改变这种行为！
>
> **alwaysRetainTaskState**： 如果栈底Activity的这个属性被设置为true，上述的情况就不会发生。 Task中的所有activity将被长时间保存。
>
> **clearTaskOnLaunch** 如果栈底activity的这个属性被设置为true，一旦用户离开Task， 则 Task栈中的Activity将被清空到只剩下栈底activity。这种情况刚好与 alwaysRetainTaskState相反。即使用户只是短暂地离开，task也会返回到初始状态 （只剩下栈底acitivty）。
>
> **finishOnTaskLaunch** 与clearTaskOnLaunch相似，但它只对单独的activity操 作，而不是整个Task。它可以结束任何Activity，包括栈底的Activity。 当它设置为true时，当前的Activity只在当前会话期间作为Task的一部分存在， 当用户退出Activity再返回时，它将不存在。

##### 4. Activity的四种加载模式详解

接下来我们来详细地讲解下四种加载模式： 他们分别是：**standard**(默认)，**singleTop**，**singleTask**，**singleInstance** 在泡在网上的日子看到一篇图文并茂的讲解启动模式的，很赞，可能更容易理解吧，这里借鉴下：

原文链接：[Activity启动模式图文详解：standard, singleTop, singleTask 以及 singleInstance](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0520/2897.html)

英文原文：[Understand Android Activity's launchMode: standard, singleTop, singleTask and singleInstance](http://inthecheesefactory.com/blog/understand-android-activity-launchmode/en) 另外还有一篇详细讲解加载模式的：[Android中Activity四种启动模式和taskAffinity属性详解](http://blog.csdn.net/zhangjg_blog/article/details/10923643)

 **先来看看总结图：** 

 ![img](https://www.runoob.com/wp-content/uploads/2015/08/50179298.jpg) 

###### 模式详解

1. standard模式

 标准启动模式，也是activity的默认启动模式。在这种模式下启动的activity可以被多次实例化，即在同一个任务中可以存在多个activity的实例，每个实例都会处理一个Intent对象。如果Activity A的启动模式为standard，并且A已经启动，在A中再次启动Activity A，即调用startActivity（new Intent（this，A.class）），会在A的上面再次启动一个A的实例，即当前的桟中的状态为A-->A。 

2. singleTop模式

 如果一个以singleTop模式启动的Activity的实例已经存在于任务栈的栈顶， 那么再启动这个Activity时，不会创建新的实例，而是重用位于栈顶的那个实例， 并且会调用该实例的**onNewIntent()**方法将Intent对象传递到这个实例中。 举例来说，如果A的启动模式为singleTop，并且A的一个实例已经存在于栈顶中， 那么再调用startActivity（new Intent（this，A.class））启动A时， 不会再次创建A的实例，而是重用原来的实例，并且调用原来实例的onNewIntent()方法。 这时任务栈中还是这有一个A的实例。如果以singleTop模式启动的activity的一个实例 已经存在与任务栈中，但是不在栈顶，那么它的行为和standard模式相同，也会创建多个实例。 

3. singleTask模式

 只允许在系统中有一个Activity实例。如果系统中已经有了一个实例， 持有这个实例的任务将移动到顶部，同时intent将被通过onNewIntent()发送。 如果没有，则会创建一个新的Activity并置放在合适的任务中。 

>  *系统会创建一个新的任务，并将这个Activity实例化为新任务的根部（root） 这个则需要我们对taskAffinity进行设置了，使用taskAffinity后的解雇：*

4. singleInstance模式

 保证系统无论从哪个Task启动Activity都只会创建一个Activity实例,并将它加入新的Task栈顶 也就是说被该实例启动的其他activity会自动运行于另一个Task中。 当再次启动该activity的实例时，会重用已存在的任务和实例。并且会调用这个实例 的onNewIntent()方法，将Intent实例传递到该实例中。和singleTask相同， 同一时刻在系统中只会存在一个这样的Activity实例。 

> 下面推荐一篇我自己找的文章，看完4个启动模式，最好顺便自己动手试试
>
> + [Activity的四种启动模式和onNewIntent()]( https://blog.csdn.net/csh86277516/article/details/79072469 )
> + [Activity详解四 activity四种加载模式](https://www.cnblogs.com/androidWuYou/p/5887807.html)

##### 5.Activity拾遗

 对于Activity可能有些东西还没讲到，这里预留一个位置，漏掉的都会在这里补上！ 首先是群友珠海-坤的建议，把开源中国的Activity管理类也贴上，嗯，这就贴上，大家可以直接用到 项目中~ 

1. 开源中国客户端Activity管理类

```java
package net.oschina.app;

import java.util.Stack;

import android.app.Activity;
import android.app.ActivityManager;
import android.content.Context;


public class AppManager {
    
    private static Stack<Activity> activityStack;
    private static AppManager instance;
    
    private AppManager(){}
    /**
     * 单一实例
     */
    public static AppManager getAppManager(){
        if(instance==null){
            instance=new AppManager();
        }
        return instance;
    }
    /**
     * 添加Activity到堆栈
     */
    public void addActivity(Activity activity){
        if(activityStack==null){
            activityStack=new Stack<Activity>();
        }
        activityStack.add(activity);
    }
    /**
     * 获取当前Activity（堆栈中最后一个压入的）
     */
    public Activity currentActivity(){
        Activity activity=activityStack.lastElement();
        return activity;
    }
    /**
     * 结束当前Activity（堆栈中最后一个压入的）
     */
    public void finishActivity(){
        Activity activity=activityStack.lastElement();
        finishActivity(activity);
    }
    /**
     * 结束指定的Activity
     */
    public void finishActivity(Activity activity){
        if(activity!=null){
            activityStack.remove(activity);
            activity.finish();
            activity=null;
        }
    }
    /**
     * 结束指定类名的Activity
     */
    public void finishActivity(Class<?> cls){
        for (Activity activity : activityStack) {
            if(activity.getClass().equals(cls) ){
                finishActivity(activity);
            }
        }
    }
    /**
     * 结束所有Activity
     */
    public void finishAllActivity(){
        for (int i = 0, size = activityStack.size(); i < size; i++){
            if (null != activityStack.get(i)){
                activityStack.get(i).finish();
            }
        }
        activityStack.clear();
    }
    /**
     * 退出应用程序
     */
    public void AppExit(Context context) {
        try {
            finishAllActivity();
            ActivityManager activityMgr= (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
            activityMgr.restartPackage(context.getPackageName());
            System.exit(0);
        } catch (Exception e) {    }
    }
}
```

###### 总结Task进行整体调度的相关操作

> - 按Home键，将之前的Task切换到后台
> - 长按Home键，会显示出最近执行过的Task列表
> - 在Launcher或HomeScreen点击app图标，开启一个新Task，或者是将已有的Task调度到前台
> - 启动singleTask模式的Activity时，会在系统中搜寻是否已经存在一个合适的Task，若存在，则会将这个Task调度到前台以重用这个Task。如果这个Task中已经存在一个要启动的Activity的实例，则清除这个实例之上的所有Activity，将这个实例显示给用户。如果这个已存在的Task中不存在一个要启动的Activity的实例，则在这个Task的顶端启动一个实例。若这个Task不存在，则会启动一个新的Task，在这个新的Task中启动这个singleTask模式的Activity的一个实例。
> - 启动singleInstance的Activity时，会在系统中搜寻是否已经存在一个这个Activity的实例，如果存在，会将这个实例所在的Task调度到前台，重用这个Activity的实例（该Task中只有这一个Activity），如果不存在，会开启一个新任务，并在这个新Task中启动这个singleInstance模式的Activity的一个实例。





下面这些暂时没有整理

##### ViewPager

切换页面用的，可以是打开应用时显示的



##### service

service：后台进程服务。有两种，service 如果阻塞2S就报错ANR，前台后台服务都可以用service

IntentService:不会有ANR问题.

可以使用AIDL实现程序之间的通讯

##### Broadcast

BroadcastReceiver广播（比如打开了wifi之类的，就会系统自动发一个广播，可以接收），标准广播类似计算机网络的ARP协议，有序广播则需要收到的继续往后传播。接受广播（动态注册、静态注册），动态注册的需要启动程序才能接受广播，静态注册则可以在不启动程序时接受广播。

Android如果4.3以上，允许程序装到SD卡上，会收不到开机广播

定义广播接收器，然后发送广播，就能接收到（不过如果需要定义服务器和客户端通信最好用service的AIDL）

广播有全局广播（其他app可以接受，其他app发全局广播，自己也会收到，存在安全问题），也有本地广播，本地广播不能静态注册接收（可以用在登录踢用户下线，比如别的手机登录了自己的账号）。

##### ContentProvider

可以获取一些系统应用、其他应用提供的数据；也可以暴露数据以向其他应用提供数据。

经常都是用来获取系统提供的信息，比如获取通讯录信息之类的，很少需要设计向其他应用提供信息。

读取存储的也是通过ContentProvider->Document Provider

日历提供者、联系人提供者、存储访问框架（文档、图片、视频、下载、。。。）

##### Intent

和组件密切相关的，用于组件间数据传输的。可以通过startActivity启动很多东西，包括一些系统组件服务

##### Application

要是一堆Activity之间传输数据，可以考虑用Application全局对象

##### Fragment

Activity里面可以放，类似Vue的组件，生命周期也受到Activity影响。

比如微信下面4个切换页面的组件（底部导航条），可以用LinearLayout+TextView或者RadioGroup+RadioButton实现；还有那种显示消息数字的底部导航条

