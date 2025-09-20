---
layout: page
title: 深度学习笔记：在自己的电脑上配置conda环境太麻烦？不妨试试Colab
permalink: /notes/colab/
---

本文是我在寻找除了自己配置Conda环境的替代方案

*   **目录**
{:toc}

---

# Google Colab 深度学习项目调试日志：运行 FastTD3

**成果：** 在 Google Colab 环境中，成功运行开源项目 FastTD3 中的 `training_notebook.ipynb` 文件。

---

## 调试过程记录

### Phase 1: 初始设置与环境准备

1.  **挂载 Google Drive**:
    *   **操作**: 使用 `google.colab.drive.mount` 将个人云端硬盘挂载到 Colab 的 `/content/drive` 目录。
    *   **目的**: 确保持久化存储项目文件，避免因会话断开而丢失代码和数据。

2.  **克隆项目与安装依赖**:
    *   **操作**:
        ```bash
        %cd /content/drive/MyDrive/FastTD3
        !pip install -r requirements.txt
        ```
    *   **遇到问题**: `pip` 报告了大量**依赖冲突 (dependency conflicts)**，例如 `opencv-python` 和 `numpy` 的版本不兼容。同时，系统警告必须**重启运行时 (Restart Runtime)**。
    *   **分析**: Colab 预装了许多库，`requirements.txt` 中指定的版本可能与这些预装库冲突。重启运行时是让新安装的库生效的**必要步骤**。
    *   **解决方案**: 遵循警告，**重启运行时**，然后重新挂载硬盘并进入项目目录。

### Phase 2: `ModuleNotFoundError` - 缺失核心环境库

1.  **第一次报错**:
    *   **错误点**: 初始化 `HumanoidBenchEnv` 环境时。
    *   **错误信息**: `ModuleNotFoundError: No module named 'humanoid_bench'`
    *   **分析**: `humanoid_bench` 是运行 `h1-walk-v0` 任务的核心依赖，但它**并未包含在**项目的 `requirements.txt` 文件中，需要手动安装。
    *   **解决方案**:
        ```bash
        !pip install git+https://github.com/carlosferrazza/humanoid-bench.git#egg=humanoid-bench
        ```
        安装后，再次**重启运行时**并重复初始设置步骤。

2.  **第二次报错**:
    *   **错误点**: 同样是在初始化环境时，但发生在 `humanoid_bench` 库内部。
    *   **错误信息**: `ModuleNotFoundError: No module named 'humanoid_bench.dmc_deps'`
    *   **分析**: 标准的 `pip install` 未能正确安装 `humanoid_bench` 的所有子模块。这是一个库打包或安装脚本的问题。需要一种更可靠的安装方式来确保所有文件都被包含。
    *   **解决方案**:
        1.  先用 `git` 将 `humanoid-bench` 的完整源码克隆到 Colab 环境中。
        2.  再使用 `pip` 的**可编辑模式 (`-e`)** 从本地克隆的文件夹进行安装，确保所有子模块都被正确链接。
        ```bash
        !git clone https://github.com/carlosferrazza/humanoid-bench.git
        !pip install -e humanoid-bench
        ```
        3.  严格执行**重启运行时**。

### Phase 3: `Session Crashed` - 资源耗尽问题

1.  **遇到问题**: 在成功解决模块缺失问题后，再次运行环境初始化代码时，Colab **会话直接崩溃 (Session Crashed)**，没有返回任何 Python 错误。
2.  **分析**:
    *   无 traceback 的崩溃通常指向底层问题，而非 Python 代码逻辑错误。
    *   最常见的原因是**内存 (RAM) 耗尽**。初始化复杂的物理模拟环境（尤其是并行环境）会瞬间消耗大量 RAM。Colab 免费版约 12GB 的 RAM 上限很容易被触及。
3.  **解决方案 (关键)**:
    1.  **彻底重置**: 使用 **“断开连接并删除代码执行程序”** 来获取一个全新的、干净的虚拟机实例。
    2.  **减小负载**: 修改创建 `args` 的代码，强制指定并行环境数量为 1，以最大限度降低内存占用。
        ```python
        args = HumanoidBenchArgs(
            # ... other args
            num_envs=1  # 关键修改！
        )
        ```

### Phase 4: 循环往复 - 理解 Colab 的“无状态”特性

1.  **再次报错**: 在解决了会话崩溃问题后，再次运行后续代码，又遇到了 `ModuleNotFoundError: No module named 'tensordict'`。
2.  **分析**: 这是对 Colab 工作模式的最终考验。执行“断开连接并删除代码执行程序”的操作，虽然解决了崩溃问题，但也**清除了所有已安装的库**。环境回到了最初的干净状态。
3.  **最终解决方案**:
    1.  整合所有安装命令到一个单元格中，形成一个完整的“环境初始化脚本”。
    2.  每次彻底重置或新开会话后，先运行这个完整的安装脚本。
    3.  **执行最终的标准流程**:
        *   运行完整的安装脚本 (`git clone`, `pip install -e`, `pip install -r`)。
        *   **重启运行时**。
        *   依次运行所有设置单元格（挂载硬盘, `%cd`, `args` 配置等）。
        *   运行项目核心代码。

---

## 核心收获与总结

1.  **Colab 是无状态的 (Stateless)**: 这是与本地 `conda` 环境最根本的区别。任何运行时的断开（无论是主动还是被动）都会导致非原生库的丢失。必须有每次都重新安装依赖的心理准备和流程设计。

2.  **重启运行时是“万能药”**: 在 Colab 中，`pip install` 之后遇到任何奇怪的问题，首选解决方案就是**重启运行时**。这能解决 90% 的库导入和版本问题。

3.  **`ModuleNotFoundError` 是路标**: 这个错误并不可怕，它清晰地指出了缺失的依赖。解决它的标准流程是：`!pip install <package_name>` -> `重启运行时`。

4.  **会话崩溃指向资源问题**: 没有错误信息的崩溃，大概率是**内存 (RAM) 耗尽**。排查方向应是减小批处理大小 (`batch_size`)、图像分辨率或并行任务数 (`num_envs`)。

5.  **Colab 的真正价值**: 尽管配置过程繁琐，但 Colab 极大地降低了深度学习的入门门槛。它将复杂的硬件和驱动配置问题，转化为了一个可以通过代码和标准流程解决的软件问题，是快速复现和学习他人代码的利器。对于拥有合格本地硬件的开发者，它更适合作为辅助工具而非主开发环境。
