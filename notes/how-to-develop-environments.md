---
layout: page
title: 深度学习笔记：配置环境时的那些坑
permalink: /notes/how-to-develop-environments/
---

在这次漫长但成果丰硕的调试过程中，我确实遇到了很多在Linux环境下进行深度学习开发时极具代表性的“坑”。将这些教训总结下来，会对未来的项目搭建非常有价值。

*   **目录**
{:toc}

---

### Linux 深度学习环境配置：从入门到专家避坑指南

本次 `humanoid-bench` 项目调试是一次典型的实战演练，它涵盖了从驱动安装、软件冲突到项目运行的全过程。以下是从中提炼出的核心注意事项与教令，遵循它们可以帮你规避未来80%以上的环境配置问题。

#### 1. 一：Conda的妙用

**教训**：
我们遇到了三大深度学习框架（PyTorch, JAX, TensorFlow）的依赖冲突问题，以及不同项目（`humanoid-bench`, `fastTD3`）对同一库（`gymnasium`）的版本要求不一。

**核心注意事项**：
- **永远不要在系统的全局Python环境中安装任何东西 (`sudo pip install ...`)。** 这是灾难的开始。
- **严格遵循“一个项目，一个环境”的原则。** 使用 `conda` 为每个独立的项目（特别是当它们来自不同作者或使用不同框架时）创建一个全新的、隔离的虚拟环境。
- **环境命名要清晰。** `conda create -n humanoid_jax python=3.10` 远比 `conda create -n my_env` 要好。

#### 2. 二：NVIDIA驱动是基础

**教训**：
我们经历了 `nvidia-smi` 成功，但OpenGL应用（MuJoCo）失败的典型场景。这揭示了NVIDIA驱动的两个面向：**计算 (CUDA) ** 和 **图形 (OpenGL)**。

**核心注意事项**：
- **区分`nvidia-driver`和`nvidia-utils`**：`nvidia-smi` 属于 `utils`，它能工作只代表驱动的管理接口正常。`nvidia-driver` 才是包含CUDA和OpenGL核心模块的完整包。
- **信任官方PPA源**：对于Ubuntu，通过 `ppa:graphics-drivers/ppa` 安装驱动，比从NVIDIA官网下载 `.run` 文件要安全、易于管理。
- **`nvidia-smi` 是你的第一道防线**：在任何调试开始前，先运行 `nvidia-smi`。如果它报错，就不要进行任何上层软件的调试，先解决驱动问题。

#### 3. 三：不要盲目相信作者的requirements文件，在开始前先将requirements文件检查一遍

**教训**：
我们分析了多个 `requirements.txt` 和 `setup.py` 文件，发现了大量直接或间接的冲突。暴力地将所有依赖合并到一个文件里是新手最常犯的错误。

**核心注意事项**：
- **学会阅读依赖文件**：
    - `==` (等于): 版本写死，最严格，也最容易冲突。
    - `~=` (兼容): `~=1.2.3` 意味着 `>=1.2.3, <1.3.0`，允许小版本修复，是很好的实践。
    - `>=` (大于等于): 比较宽松，但有风险。
- **`extras_require` 是高级武器**：当遇到一个项目支持多种框架（如 `humanoid-bench` 支持JAX, PyTorch等）时，通过 `pip install .[jax]` 或 `pip install .[torch]` 来按需安装，能保持环境的纯净。
- **`ModuleNotFoundError`**：这是最简单的错误，通常意味着你只是忘了安装某个库。直接 `pip install [库名]` 就能解决。

#### 4. 四：一个庞大的项目无法一蹴而就，学会分析错误日志是关键

**教训**：
从最严重的`程序崩溃`，到 `AttributeError`，再到 `gladLoadGL error`，最后到 `TypeError`，每一个的错误日志都代表不同的含义，不要一看到红色报错就发怵。

**核心注意事项**：
- **永远看最后一行**：Traceback信息很长，但**最后一行**通常是最直接的错误类型和描述（如 `ModuleNotFoundError: No module named 'moviepy'`）。
- **学会识别错误类型**：
    - `ModuleNotFoundError`: 缺少库。
    - `AttributeError`: 代码笔误或版本不匹配。
    - `TypeError`: 函数用法错误，通常是库版本冲突。
    - `mujoco.FatalError`: 硬件/驱动层面的问题。
- **利用搜索引擎或者AI**：不认识的错误信息，可以复制到Google搜索或者发给AI。

**其他注意事项**：

- **创建别名 (Alias) 简化操作**：将那串长长的环境变量命令通过 `alias prun='...'` 保存到你的 `.bashrc` 中，能极大地提升效率。
- **miniconda和anaconda的选择**：
Anaconda：这是一个巨大的软件包发行版。预装了超过250个在数据科学、机器学习领域最常用、最流行的“零件”（软件包），比如 a, b, spyder, jupyter notebook 等。它就像一个**“精装修拎包入住”的公寓**，所有家具家电都给你配齐了。

Miniconda：这是 Anaconda 的“迷你版”。它也包含了 conda 工具箱，但是几乎没有预装任何软件包（只有一个最基础的 a）。它就像一个**“毛坯房”**，只给了你最基本的水电，你需要什么家具家电（软件包）都得自己亲自去挑选和安装。
---

通过这次实战，我亲身体验并克服了深度学习环境配置中最常见、也最令人头疼的几大类问题。并将这些教训内化为你的开发习惯。
