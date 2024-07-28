# Godot(官方教程)-学习笔记

> [Introduction — Godot Engine (stable) documentation in English](https://docs.godotengine.org/en/stable/about/introduction.html)
>
> [Learn Godot's GDScript From Zero by GDQuest, Xananax (itch.io)](https://gdquest.itch.io/learn-godot-gdscript)
>
> [Godot文件- 主 分支 — Godot Engine latest 文档 (osgeo.cn)](https://www.osgeo.cn/godot/index.html)
>
> [Vignette - Godot Shaders](https://godotshaders.com/shader/vignette/)

1. Godot默认子节点ready之后，才会ready父节点。如果需要在子节点中直接或间接调用到父节点的onready变量，就会报错。可以通过`await owner.ready ` 先等待父节点ready
2. `@export`变量在`__init__`之后，`_ready()`方法之前初始化，在`_ready()`可以安全访问
3. `@onready`变量在`_ready()`方法中由Godot自动初始化
4. 普通变量的初始化在`@export`变量之前
5. 初始化顺序：`普通var` > `@export` > `@onready`
6. Godot制作2D像素游戏，纹理大小Rect宽高最好设置为偶数，避免图片在相机移动时发生抖动
7. input输入事件传播的调用链（注意，输入事件的传播和输入状态查询`Input.get_axis("move_left")`是两套系统）：`_input()`>`_gui_input()`>`_shortcut_input()`>`unhandled_key_input()`>`_unhandled_input()`>其他逻辑。按照上述顺序，<u>前面的函数没有处理逻辑则继续向后检测处理逻辑</u>。

8. 