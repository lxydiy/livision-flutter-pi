# Flutter-Pi 离屏渲染同步对照

`0004-diagnostic-gl-finish-before-scanout.patch` 在 EGL/GBM 输出路径中增加
可选的 `glFinish()`：成功 `eglSwapBuffers()` 后、锁定 GBM 前缓冲并交给
KMS 之前执行。它等待当前 Flutter 渲染 context 的既有 GL 命令完成，包含
swap 触发的提交；不修改内核，不改变 KMS 翻页方式，不处理 Vulkan 路径。

## 应用与构建

现有 `build.sh` 会自动应用 `patches/*.patch`，因此包含本补丁。
注意它调用的 `scripts/apply_patches.sh` 会 hard reset 子模块到
`origin/master` 并 clean；有本地修改时应在干净的独立副本构建。

也可以在已经应用 0001–0003 的源码上单独应用：

```bash
cd flutter-pi
git apply --check ../patches/0004-diagnostic-gl-finish-before-scanout.patch
git apply ../patches/0004-diagnostic-gl-finish-before-scanout.patch
cd ..
cmake -S flutter-pi -B build
cmake --build build -j2
```

按现有流程安装/运行构建产物，确认实际运行的是这次构建的二进制。

## A/B 对照

环境变量只在值恰好为 `1` 时开启；未设置或设为 `0` 都关闭。
每次切换都重新启动 Flutter-Pi，其他参数保持一致。

```bash
# A：原始异步路径
FLUTTER_PI_DEBUG_GL_FINISH=0 flutter-pi --release /path/to/bundle

# B：等待 GPU 完成
FLUTTER_PI_DEBUG_GL_FINISH=1 flutter-pi --release /path/to/bundle

# 若使用 sudo，显式把变量传给目标进程
sudo env FLUTTER_PI_DEBUG_GL_FINISH=1 flutter-pi --release /path/to/bundle
```

开启后，第一次成功执行等待会输出：

```text
[diagnostic] GL finish ACTIVE: after EGL swap, before GBM/KMS handoff.
```

补丁核对当前 EGL display、Flutter 渲染 context 和 EGL draw surface。
不匹配时不切换 context，不在其他 context 上制造无效等待，而输出
`GL finish SKIPPED`；出现该信息的运行不能作为有效的开启对照。
没有 ACTIVE 信息也不能认为等待已执行。日志按 surface 状态变化输出，
避免每帧打印。等待本身也有 `diagnostic_glFinish` tracer 区间。

测试程序中先选择数量并 Prepare，待准备完成后 Show together；随后降低
显示数量再恢复，复用已有图像。观察黄色顶层标记、UI tick 和中央转圈。
UI tick 不是已显示帧号；隐藏对象的模糊中间纹理不保证常驻显存。

若 B 消除异常，优先检查 fence、提交和缓冲复用，但不能仅凭这一结果确认
同步 bug：glFinish 同时降低并行度、改变负载和时序。此补丁是诊断开关，
不是修复，也不等待其他 GL context 中独立的 LiVision 纹理上传工作。

## 本地验证范围

补丁 0001–0004 已在隔离源码快照上按顺序应用验证，基线为本地
`origin/master`：`8b86c52a004ec2ca7f98a65e5079219679002c61`。
本次未在目标 Linux/ARM 环境编译或运行 Flutter-Pi；Windows Flutter
测试和构建不构成 Flutter-Pi 实机验收。
