# 具身智能开发环境搭建日志 - Day 1

**日期:** 2025-11-24
**目标:** 从零搭建基于 Ubuntu 22.04 + ROS 2 Humble 的具身智能开发环境
**核心成果:** 完成系统升级，安装 ROS 2 核心，跑通基础 Demo。

-----

## 1. 操作系统升级与基础配置

### 1.1 系统升级 (Ubuntu 20.04 -> 22.04)
* **背景**: ROS 1 Noetic 即将停止维护，ROS 2 Humble (LTS) 需要 Ubuntu 22.04 环境。
* **操作**: 使用 `do-release-upgrade` 进行在线升级。
* **遇到问题**: 升级过程中卡在 `Installing the firefox snap`，下载速度极慢。
* **解决方案**:
    * **原因**: Snap Store 服务器在国外，直连速度极慢。
    * **解决**: 开启 VPN 加速，或保持耐心等待（约 30-60 分钟）。**切勿强行中断**。

### 1.2 输入法配置 (Fcitx5)
* **背景**: 搜狗输入法在 Ubuntu 22.04 下依赖旧版 Qt 库，极不稳定（闪退/无候选框）。
* **解决方案**: 彻底卸载搜狗，拥抱原生 **Fcitx5**。
* **关键命令**:
    ```bash
    # 1. 清理旧版
    sudo apt purge sogoupinyin fcitx* -y
    rm -rf ~/.config/sogoupinyin ~/.config/fcitx

    # 2. 安装 Fcitx5
    sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk4 kde-config-fcitx5 -y

    # 3. 切换框架并重启
    im-config -n fcitx5
    ```

-----

## 2. ROS 2 Humble 安装 (核心环节)

### 2.1 软件源配置 (VPN vs 国内源)
* **核心原则**:
    * **有 VPN**: 推荐使用 **ROS 官方源** (`packages.ros.org`) + `rosdep` 官方 GitHub 源。
    * **无 VPN**: 推荐使用 **清华/阿里镜像源** + `rosdepc`。
    * **禁忌**: 开着 VPN 去访问清华源（会导致 HTTP 403 Forbidden 报错）。

### 2.2 依赖管理 (Rosdep)
* **遇到问题**: 运行 `rosdep update` 时全屏报错 `HTTP Error 403: Forbidden`。
* **原因**: 配置文件中残留了清华源地址，但系统开启了 VPN，被清华源防火墙拦截。
* **解决方案**:
    ```bash
    # 1. 删除所有旧配置
    sudo rm /etc/ros/rosdep/sources.list.d/*.list

    # 2. 重新初始化官方源 (需保持 VPN 开启)
    sudo rosdep init
    rosdep update
    ```

-----

## 3. 常见报错汇总 (Troubleshooting)

### 3.1 缺少 C++ 标准库
* **现象**: CMake 报错 `/usr/bin/ld: 找不到 -lstdc++`。
* **原因**: 系统升级后，GCC/Clang 的基础开发包不完整。
* **解决**:
    ```bash
    sudo apt install build-essential libstdc++-12-dev
    ```

### 3.2 Anaconda 环境冲突 (BOSS 级问题) ⚠️
* **现象**: `colcon build` 失败，报错 `ModuleNotFoundError: No module named 'em'` 或 `'catkin_pkg'`。
* **原因**: 终端默认激活了 Conda `(base)` 环境，ROS 2 编译工具错误调用了 Conda 的 Python 解析器，而非系统 Python。
* **解决方案**:
    1.  **退出环境**: `conda deactivate`
    2.  **清理缓存 (必须)**: `rm -rf build install log` (不删缓存会一直报错)
    3.  **永久关闭自启**: `conda config --set auto_activate_base false`
    4.  **重新编译**: `colcon build`

-----

## 4. 最终成果与项目结构

### 4.1 项目架构
```text
~/Embodied_AI_Project/          # 总项目目录
    ├── ros2_ws/                # ROS 2 工作空间
    │    ├── src/               # 源代码 (手动创建)
    │    │   └── robot_brain/   # 第一个 Python 包
    │    ├── build/             # 编译中间件 (自动生成)
    │    ├── install/           # 编译产物 (自动生成)
    │    └── log/               # 日志