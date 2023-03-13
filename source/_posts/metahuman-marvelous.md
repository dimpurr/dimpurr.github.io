---
title: 将带动作捕捉绑定的 Unreal 5 Metahuman 导入 Marvelous Designer 进行布料模拟
tags: 
- Unreal
date: 2023-03-13 19:43:32
---

前提：已经制作好了一个通过 IK Retargeting 重定向绑定好骨骼动画的 metahuman 模型。

考虑到除了衣服布料之外还需要做场景布料模拟，调研了 UE 一些场景的布料解决方案。

1. UE 原生的 Niagara 引擎中有自己的布料系统，但是都非常的菜。或许足以用在游戏场景中，但是对影视效果来说有点太过不真实。
2. NVIDIA 提供了 Apex Cloth 和 NvCloth 两个插件，但前者主要是做衣服模拟，后者比较通用但是十分底层。

目前看，场景布料模拟最好的方案反而是直接用 Houdini 完成后导出 ABC 。同理，调研结果是 Marvelous Designer 是公认最好的布料模拟系统，所以这里最好的工作流是：

1. 在 UE5 中完成 metahuman 的骨骼绑定，并导出 FBX
2. 在 MD 中导入 metahuman FBX ，完成衣服贴片和动作导出
3. 在 UE5 中再次导入

以下是踩坑详情。

### UE5 Metahuman 导出

<!-- more -->

第一个问题是新版本的 Metahuman Creator 的身体 Mesh 都不包含头部（颈部以上）的部分，而且没有一个简单的工作流可以让你不经过 Blender 或者 Maya 将他们合并到一起。显然，在一个没有脖子的身体上进行布料模拟是不可能的，因为衣服会陷进脖子区域的孔洞。解决方案是下载老版本的“带头”模型：

https://drive.google.com/drive/folders/1SNLa38CeMLmYq8KLQn67f6xFxM_7zkpa

所有模型按性别和体型分配，在你的 metahuman 模型找到 body 对应的 mesh 名字并下载。拖拽这个 FBX 并导入。

{% asset_img "import-metahuman-full-fbx.png" 350 "Import Metahuman Full FBX" %}

确保你勾选了 Import Mesh 导入网格体的复选框，以及在骨骼中选中 metahuman_base_skel 。其他的设置都无关紧要，保留默认即可。

随后双击打开，在左边资产详情中找到 骨骼网格 - 后期处理动画蓝图，在列表中找到与你导入的体型对应的 BP Animation 并选中。点击左上角保存并离开这个编辑界面。

{% asset_img "mesh-anime-bp.png" 350 "Import Metahuman Skeleton Mesh Anime BP" %}

再次打开你的 IK Retargeting 动画绑定器（如果你没有采用 IK 重新映射，直接打开你的动画文件），找到 Preview Settings 附近的“目标”，在目标预览网格体中下拉选择你导进来的 _FULL mesh 。

{% asset_img "preview-target-mesh.png" 150 "Import Metahuman Skeleton Mesh Anime BP" %}

在这里我的情况会有提醒是否绑定垂直偏移的提醒，原则上应该选是，但我这里可能存在骨骼不一致的情况会导致人物乱飞，待稍后 debug ：

{% asset_img "retarget-root.png" 350 "Import Metahuman Skeleton Mesh Anime BP" %}

然后在右下角选择对应的动画序列，点导出选定动画，然后双击打开新导出的动画。

另外，如果你是直接在动画而非 IK Retargeting 中编辑，记得点击 Apply to Assets 。

无论你用哪种方式完成，现在都应该在一个动画的编辑界面，并且预览 Mesh 已经被替换为了带头部的 Mesh 。点击保存动画即可。

我的 UE 在这里会孜孜不倦的 Crash ，如果在不保存 anim 的情况直接点开 anim 文件，报错显存炸了。猜测是 metahuman 的衣服精度太高。不点开 anim 文件直接保存仍然报错，查看报错是内存炸了。尝试在 Windows 系统设置中设置系统托管的分页文件（自动虚拟内存）然后重启。在保存时打开内存监控面板，然后眼睁睁看着内存从44GB涨到57GB……并祈祷它不会炸。（第一次在生产环境单任务把 64G 内存接近吃满）就我的情况，考虑到直接导入这个动画本身就要花将近 5 分钟，因此觉得可以耐心等待 5-10 分钟。然而还是不行。

（未完待续）


本文主要参考了以下教程：

- HOW TO import Marvelous Designer clothes for a MetaHuman (UE5 & MD10)
 https://www.youtube.com/watch?v=gH_kExD4fFM