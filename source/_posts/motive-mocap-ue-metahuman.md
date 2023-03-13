---
title: >-
  将 OptiTrack Motive 导出的 FBX mocap 动作捕捉骨骼动画利用 IK (FK) Retargeting 重映射绑定到 Unreal
  Metahuman 上
tags:
  - UE
date: 2023-03-13 21:09:45
---


拿到 FBX 格式的动捕导出动画，通过 VSCode 暴力打开二进制查看头部信息发现如下字样：

```
Document	Motive Scene
Title		Motive Scene
Subject		Exported from Motive
```

调查后发现这是一个商业收费动捕软件。

### 向场景中新增 Metahuman

新建一个空游戏项目。在左上角 Add 找到 Quixel Bridge 。找到左侧列表的头像按钮，选中 MetaHuman Presets ，找一个合适的模型。

<!-- more -->

原则上还应可以在左下角下拉列表中选中低模，但不知道为什么只有高模选项。点击下载。开始下载前一般会要求你再登录一次。随后在下载任务管理器中查看大小，一般在 1-2 GB 之间不等，测试时尽量选一个偏小的不要给自己找罪受。还有个恐怖故事，就是有时候你会下载完了发现这个 metahuman 没有身体，衣服的部分是空的， body 模型只有裸露的手臂和脚踝，还有的就是骨骼不知道为什么没有头和手臂部分的骨骼 …… 目前最正常的是 Amelia 。

下载完成后点击 Add ，人物就被添加到场景了。这时候会提示你要启用一些新插件，全部启用然后重启 UE 生效。

### 修改 Mocap FBX 并导入 UE

跳过调研过程，直接说结论：Motive 默认导出的 FBX 文件结构有些问题。用 Blender 新建空文件，导入 _body 后即可看到文件结构如下：

{% asset_img "original-skel-struct.png" "Metahuman Skeleton Structure" %}

接下来我们要做的就是把 Motive 导出的 FBX 文件从原有结构：

{% asset_img "motive-skel-origin.png" 350 "Motive Skeleton Orginial" %}

修改成这个结构：

{% asset_img "motive-skel-struct.jpg" 450 "Motive Skeleton Structure" %}

具体操作是，按照图标对应层级，

1. 将 Cube 按住 Shift 拖进 skeleton 下
2. 把原来的 hips 骨骼节点重命名为 root

即可。

该解决方案的参考来源： https://forums.unrealengine.com/t/optitrack-motive-to-ue4-how-to-create-root-bone/359610 (`i just added mesh to skeleton(just cube) and named the whole skeleton as “root” and it exported/imported just fine`)

在 Blender 中再次导出编辑完毕的内容为 FBX ，然后拖进 UE 。

{% asset_img "import-mocap-edited-fbx.png" 350 "Import Edited Mocap FBX" %}

Skeleton 骨骼选择 metahuman_base_skel 或者不选， Advanced 下面的 Update Skeleton Ref 可选， Use T0 As Fef Pose 按实际情况选你导入的是不是 TPose （无法预料这个行为）。一定要记得勾选底部的导入动画（Animation - Import Animations）。其他默认。

{% asset_img "importing-edited-fbx.png" "Importing Edited Mocap FBX" %}

此时会有一个不短的导入时间，耐心等待完成。

### 分别新建 metahuman 和 mocap 的 IK Rig ，进行 FK 链绑定

右键新建 - 动画 - IK Rig - IK Rig 。首先选择类似 f_med_nrw_body 字样的身体，完成创建，这里示意命名为 metahuman 。然后再新建一个 IK Rig ，选中刚导入进来的 mocap skeleton mesh ，完成创建，这里示意命名为 mocap 。

然后双击打开两者，可以并排窗口，观察骨骼结构并一一为其赋予名字相同的 FK 节点。

首先在 root 或者 skeleton 上右键 Set Retarget Root 。然后在接下来要设置 FK 节点的端点，右键 New Retarget Chain from Selected Bones 。在弹窗里选中 No Goal 不要设置 IK 解算器，

{% asset_img "metahuman_fk_struct.png" 250 "Metahuman FK Struct" %}

### 新建 IK Retargeting 重绑定器

右键新建 - 动画 - IK Rig - IK Rig

（未完待续）