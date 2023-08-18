---
title: 在 Windows 下使用 PyTorch 运行 DMPFold2 进行蛋白质折叠预测
date: 2023-05-26 22:35:40
tags: 
- Machine Learning
- Bioinfo
- Media Art
slug: win-protein-pdb
---


使用 Anaconda 之前配置好的 PyTorch 环境，这里不再赘述，可以自行寻找相关教程。

```sh
pip install dmpfold
```

安装后直接运行 `dmpfold` 会提示 `'dmpfold' 不是内部或外部命令，也不是可运行的程序或批处理文件。`。最后发现的解决方案是通过 `python C:\Users\myccy\anaconda3\envs\pytorch\Scripts\dmpfold` 的方式运行。

尝试在 macOS (M1 Max) 上运行的时候，还碰到了一个 Bug :

```
OMP: Error #15: Initializing libiomp5.dylib, but found libomp.dylib already initialized.
OMP: Hint This means that multiple copies of the OpenMP runtime have been linked into the program. That is dangerous, since it can degrade performance or cause incorrect results. The best thing to do is to ensure that only a single OpenMP runtime is linked into the process, e.g. by avoiding static linking of the OpenMP runtime in any library. As an unsafe, unsupported, undocumented workaround you can set the environment variable KMP_DUPLICATE_LIB_OK=TRUE to allow the program to continue to execute, but that may cause crashes or silently produce incorrect results. For more information, please see http://www.intel.com/software/products/support/.
```

貌似这个 Bug 多发于 macOS Apple Silcon 中的 TensorFlow 和 PyTorch ，使用 `KMP_DUPLICATE_LIB_OK=TRUE dmpfold -i input.aln -n 0 -m 0 > fold.pdb` 这种方式运行可以绕过。更复杂的解决方式是重新安装 `mkl` 依赖或者修改 import 顺序，我没有尝试。

按上述方法运行后会报错 `[1]    26296 segmentation fault  KMP_DUPLICATE_LIB_OK=TRUE dmpfold -i *.aln -n 0 -m 0 > fold.pdb` ，不清楚问题出在哪，遂放弃。