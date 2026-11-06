# Demo-RM-C

基于 STM32F407IGH6 的 C11 固件教学入门工程，使用 STM32CubeMX、ARM GNU Toolchain 和 CMake/Ninja 构建，通过 OpenOCD 烧录。

## 环境要求

Windows、Linux 和 macOS 均使用下列工具，并将其可执行文件所在目录加入 `PATH`：

- CMake ≥ 3.22。
- Ninja。
- ARM GNU Toolchain：`arm-none-eabi-gcc`、`arm-none-eabi-g++` 等交叉编译工具，需包含目标平台的 C/C++ 标准库。
- OpenOCD：烧录时需要，下述配置以 OpenOCD 0.12.0 为基准。
- clangd：可选，用于编辑器补全、诊断和跳转。

可使用 STM32CubeCLT 提供的 ARM GNU Toolchain。以下命令均在项目根目录执行，适用于 PowerShell、Bash 和 Zsh。

```sh
cmake --version
ninja --version
arm-none-eabi-gcc --version
```

## 配置与编译

项目已在 `CMakePresets.json` 中配置 Ninja 生成器、交叉编译工具链和构建目录。

```sh
# 查看可用的配置和构建预设
cmake --list-presets=all

# Debug：首次使用先配置，再编译；后续修改源码通常只需执行编译命令
cmake --preset Debug
cmake --build --preset Debug --parallel

# Release：使用独立目录，配置后再编译
cmake --preset Release
cmake --build --preset Release --parallel
```

预设名称区分大小写。

| 预设 | 构建目录 | 编译选项 | 主要产物 |
| --- | --- | --- | --- |
| `Debug` | `build/Debug` | `-O0 -g3` | `Demo-RM-C.elf`、`Demo-RM-C.map` |
| `Release` | `build/Release` | `-Os -g0` | `Demo-RM-C.elf`、`Demo-RM-C.map` |

产物位于各自的构建目录中。ELF 文件用于烧录，链接映射文件（`.map`）用于查看程序的内存布局。

## OpenOCD 烧录

构建成功后，在项目根目录按实际下载器选择一条命令执行。

```sh
# Debug — CMSIS-DAP（默认）
openocd -f openocd/cmsis-dap.cfg -c "program {build/Debug/Demo-RM-C.elf} verify reset exit"

# Debug — ST-Link
openocd -f openocd/stlink.cfg -c "program {build/Debug/Demo-RM-C.elf} verify reset exit"

# Debug — J-Link
openocd -f openocd/jlink.cfg -c "program {build/Debug/Demo-RM-C.elf} verify reset exit"

# Release — CMSIS-DAP（默认）
openocd -f openocd/cmsis-dap.cfg -c "program {build/Release/Demo-RM-C.elf} verify reset exit"

# Release — ST-Link
openocd -f openocd/stlink.cfg -c "program {build/Release/Demo-RM-C.elf} verify reset exit"

# Release — J-Link
openocd -f openocd/jlink.cfg -c "program {build/Release/Demo-RM-C.elf} verify reset exit"
```

## 常用 CMake 命令

以下以 Debug 为例；需要操作 Release 时，将预设名或目录中的 `Debug` 替换为 `Release`。这些命令要求对应目录已完成配置。

```sh
# 限制并行编译任务数
cmake --build --preset Debug --parallel 4

# 查看实际编译、链接命令
cmake --build --preset Debug --verbose

# 清理编译产物，保留 CMake 配置
cmake --build --preset Debug --target clean

# 先清理，再完整编译
cmake --build --preset Debug --clean-first --parallel

# 查看可用构建目标
cmake --build --preset Debug --target help

# 查看 CMake 缓存变量
cmake -LA -N build/Debug

# 按构建目录编译，与使用对应的构建预设等价
cmake --build build/Debug --parallel
```

使用 CMake ≥ 3.24 时，可通过 `--fresh` 清除 CMake 缓存和 `CMakeFiles` 后重新配置，再执行编译。它与清理编译产物的 `--target clean` 用途不同。

```sh
cmake --preset Debug --fresh
cmake --build --preset Debug --parallel
```

更换编译器或工具链安装位置后，应使用新的构建目录，或清除旧配置后重新配置。

## clangd

项目已启用 `CMAKE_EXPORT_COMPILE_COMMANDS`。运行 `cmake --preset Debug` 后，会生成 `build/Debug/compile_commands.json`；根目录的 `.clangd` 默认读取该文件，复用工程的头文件路径、宏定义和 ARM 编译参数。

在编辑器的 clangd 启动参数中加入以下一项，使其能够查询 ARM GCC 的目标架构和标准库头文件路径：

```text
--query-driver=**/arm-none-eabi-gcc*
```

这是 clangd 的启动参数，需配置在编辑器或 LSP 客户端中。命令行也可检查配置是否生效：

```sh
clangd --check=Core/Src/main.c "--query-driver=**/arm-none-eabi-gcc*"
clangd --check=Drivers/STM32F4xx_HAL_Driver/Inc/stm32f4xx_hal_def.h "--query-driver=**/arm-none-eabi-gcc*"
```

单独打开 HAL 模块头文件或 `stm32f4xx_hal_conf.h` 时，`USE_HAL_DRIVER` 会使 `stm32f4xx.h` 提前递归包含 HAL，可能在 `HAL_StatusTypeDef` 定义前使用该类型。`.clangd` 仅在独立解析这些头文件时移除该宏，避免这类包含顺序诊断；解析 `.c` 文件和实际构建仍使用完整的工程宏定义。

如需按 Release 配置索引，先运行 `cmake --preset Release`，再将 `.clangd` 的 `CompilationDatabase` 改为 `build/Release`。更换编译环境后重新生成编译数据库，并重启编辑器中的 clangd。

参考：[CMake 命令行文档](https://cmake.org/cmake/help/latest/manual/cmake.1.html)、[clangd 配置](https://clangd.llvm.org/config)、[clangd 交叉编译器头文件查询](https://clangd.llvm.org/guides/system-headers)。
