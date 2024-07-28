# GDScript基础-学习笔记

> [GDScript - GDScript 基础 - 《Godot 游戏引擎 v3.5 中文文档》 - 书栈网 · BookStack](https://www.bookstack.cn/read/godot-3.5-zh/f9c328f5c1cf8c98.md)
>
> 以前学过JavaScript和Python，所以这里就不特地写全面的笔记了。

1. GDScript支持鸭子类型

2. GDScript继承Refrence的引用类型在计数为0时会自动释放，继承Object则需要手动管理内存

3. 一般开发时最基础的类只用到`RefCounted`，再往下的类需要手动内存管理，比较繁琐
4. 