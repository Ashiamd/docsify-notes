# Android开发基础教程（2019）学习笔记

> 二话不说，贴其中一个视频的链接，其他的也是同一个up主投递
>
> https://www.bilibili.com/video/av52405925/?spm_id_from=333.788.videocard.0

#### 2. "Hello World"

#### 3.界面布置ConstraintLayout(1)

+ ConstraintLayout使用魔法棒，自动部署布局

#### 4. 界面布置ConstraintLayout(2)

+ Guideline、Barrier和Group的使用
  + Guideline：实际界面不显示，但是可以把元素位置锁定到其上，方便设计（主要掌握百分比辅助线使用）
  + Barrier：实际不显示，遇到放入其中的元素会停下，变成以另一个作为拖动边界的条件
  + Group：可以统一对所属的元素进行属性设置

#### 5. Activity LifeCycle

+ onCreate()-> onStart()-> onResume() -> onPause() -> onStop() -> onDestroy() 
+ 调用finish()会触发onDestroy(),切换为横屏也会触发onDestroy，重新创建Activity
+ 需要在onDestroy()之前释放资源，避免内存泄漏
  + 自己尝试了下，回到桌面会执行onPause(),onStop().
  + 关闭程序Log没显示执行onDestroy()
  + 从桌面回去程序执行onRestart(),onStart()，onResume()

+ 官方Activity生命周期图

   ![img](https://developer.android.google.cn/guide/components/images/activity_lifecycle.png) 

#### 6. XXX

