# TRex ARM64 (aarch64) 编译与补充依赖操作指南

本文档记录了 TRex 在 ARM64 (aarch64) 架构下的 DPDK 集成修复情况，以及后续在真实 ARM64 环境中进行编译与运行所需补充的文件与操作步骤。

---

## 一、 背景与已修复内容

TRex 采用源码内嵌 DPDK 并通过 Waf 构建系统进行管理的方式。当前 `arm` 分支已完成以下核心底层适配：

1. **DPDK 25.07 aarch64 配置快照补齐**：
   * 创建了 `src/pal/linux_dpdk/dpdk_2507_aarch64/` 目录。
   * 补全了 `rte_config.h` 和 `rte_build_config.h`，定义了 `RTE_ARCH arm64`、`RTE_ARCH_ARM64` 及默认 Neon/CRC32/AES 指令集标志，并解除了 x86 专有的宏定义。
2. **构建脚本 `linux_dpdk/ws_main.py` 修复**：
   * 修正了 `DPDK_FLAGS` 中的头文件包含路径为 `dpdk_2507_aarch64`，并补齐 `-DALLOW_INTERNAL_API` 与 `-DABI_VERSION="25.2"`。
   * 更新了 EAL arm 源码目录，由过时的 `lib/eal/common/arch/arm/` 改为 `lib/eal/arm/`，并补充了 `rte_hypervisor.c`、`rte_mmu.c` 和 `rte_power_intrinsics.c`。
   * 分离了通用组 `dpdk_src` 中的 x86 SSE 源码，为 aarch64 接入 Neon 向量化实现（`bnxt_rxtx_vec_neon.c`、`iavf_rxtx_vec_neon.c`、`i40e_rxtx_vec_neon.c`）。
   * 补齐了 aarch64 下的 Intel i40e PMD 驱动依赖和 Mellanox (mlx5) 构建分支。
   * 解除 GCC 编译全局硬编码的 x86 TSX 指令集标志 `-mrtm`，将其限制在 Intel x86 平台生效，避免 ARM GCC 报 `unrecognized command-line option '-mrtm'` 错误。
   * 修复了安装软链接时目标目录 `scripts/so/aarch64` 缺失的问题。
3. **高精度时钟抽象 `src/os_time.h` 优化**：
   * 在读取 `cntvct_el0` 虚拟时钟时增加了 `isb` 内存屏障保障指令顺序。
   * 动态读取 `cntfrq_el0` 获取真实的 CPU 基础时钟频率，避免了写死 3GHz 带来的计时严重失真。

---

## 二、 后续需补充编译与放置的文件清单

在 ARM64 目标机上完成完整编译与运行前，需要准备并补充以下动态库和文件：

### 1. ZeroMQ 动态库（运行必须）

TRex 的 C++ 核心与 Python RPC 客户端之间重度依赖 ZeroMQ。当前仓库历史遗留的是 2018 年的 `libzmq.so.3`，需要替换为 ZeroMQ 4.3.x (`libzmq.so.5`)。

| 目标文件位置 | 作用 | 说明 |
| :--- | :--- | :--- |
| `external_libs/zmq/aarch64/libzmq.so` | 编译期链接与静态分发 | 放置编译生成的 `libzmq.so` (或软链接到 `libzmq.so.5`) |
| `scripts/so/aarch64/libzmq.so.5` | 运行期动态加载 | TRex 可执行程序启动时通过 rpath 加载 |
| `scripts/external_libs/pyzmq-ctypes/zmq/arm/libzmq.so` | Python 客户端调用 | 供 TRex 控制台脚本（trex-console、Scapy RPC 服务）调用 |

#### 编译与部署步骤：
在 ARM64 机器上执行：
```bash
# 下载并编译 ZeroMQ
git clone https://github.com/zeromq/libzmq.git -b v4.3.5
cd libzmq
./autogen.sh
./configure --prefix=/tmp/zmq_arm64 --without-docs
make -j$(nproc) && make install

# 回到 trex-core 根目录，拷贝动态库
cp /tmp/zmq_arm64/lib/libzmq.so.5* external_libs/zmq/aarch64/
ln -sf libzmq.so.5 external_libs/zmq/aarch64/libzmq.so

mkdir -p scripts/so/aarch64
cp /tmp/zmq_arm64/lib/libzmq.so.5* scripts/so/aarch64/

mkdir -p scripts/external_libs/pyzmq-ctypes/zmq/arm
cp /tmp/zmq_arm64/lib/libzmq.so.5 scripts/external_libs/pyzmq-ctypes/zmq/arm/libzmq.so
```

---

### 2. Mellanox / OFED 动态库（使用 CX4/CX5/CX6 网卡时必须）

若在 ARM64 机器上使用 Mellanox (NVIDIA) 网卡，TRex 需要 `libibverbs.so` 和 `libmlx5.so`。

有两种处理方式（推荐方式 A）：

* **方式 A（推荐）：直接使用系统或官方 OFED 驱动**
  在 ARM 服务器上安装 Mellanox OFED 或系统 `rdma-core` 包：
  ```bash
  # Ubuntu / Debian
  apt-get install -y libibverbs-dev librdmacm-dev libmnl-dev
  # CentOS / RHEL / OpenEuler
  yum install -y libibverbs-devel librdmacm-devel libmnl-devel
  ```
  在配置 TRex 时带上 `--with-ofed` 参数：
  ```bash
  cd linux_dpdk
  ./b configure --with-ofed
  ```

* **方式 B：补充内嵌 prebuilt 二进制**
  若需要独立打包运行，可将编译好的 OFED 运行时放入以下路径：
  * `external_libs/ibverbs/aarch64/libibverbs.so`
  * `external_libs/ibverbs/aarch64/libmlx5.so`
  * `external_libs/ibverbs/aarch64/include/`（包含 infiniband 头文件）

---

### 3. ARM64 用户态高精度 PMU 周期计数器（可选性能项）

#### 说明：
* 默认情况下，TRex 使用 ARM 系统通用虚拟计数器 `cntvct_el0`。它的频率通常在 **25 MHz ~ 100 MHz**，足以保证日常大部分吞吐量和功能测试正常运行。
* 如果对纳秒级发包延迟测试、极高精度抖动测量有严苛要求，可启用 CPU 全频周期计数器 `pmccntr_el0`。
* Linux 内核默认禁止普通用户态程序（EL0）直接读取 `pmccntr_el0`（直接读取会触发 `SIGILL` 非法指令崩溃）。若要启用，需加载一个微型内核模块开启用户态权限。

#### 开启步骤：
编写微型内核模块 `enable_arm_pmu.c`：
```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init enable_pmu_init(void) {
    /* 允许 EL0 用户态读取 PMCCNTR_EL0 计数器 */
    asm volatile("msr pmuserenr_el0, %0" : : "r"(1ULL | (1ULL << 2)));
    asm volatile("msr pmcntenset_el0, %0" : : "r"(1ULL << 31));
    printk(KERN_INFO "ARM64 PMU: User-space PMCCNTR_EL0 access enabled\n");
    return 0;
}

static void __exit enable_pmu_exit(void) {
    asm volatile("msr pmuserenr_el0, %0" : : "r"(0ULL));
    printk(KERN_INFO "ARM64 PMU: User-space PMCCNTR_EL0 access disabled\n");
}

module_init(enable_pmu_init);
module_exit(enable_pmu_exit);
MODULE_LICENSE("GPL");
```
加载该模块后，在编译 TRex 时定义宏 `-DUSE_HIGH_RES_CYCLES_COUNTER` 即可享受 GHz 级别的硬件时钟精度。

---

## 三、 编译与验证命令

在补充完成上述第 1 项（ZeroMQ 库）后，即可在 ARM64 设备上进行编译：

```bash
# 1. 切换到构建目录
cd linux_dpdk

# 2. 配置 (若不使用 Mellanox 网卡，可添加 --no-mlx 避免检查 OFED)
./b configure --no-mlx
# 或使用系统 OFED：
# ./b configure --with-ofed

# 3. 编译
./b build -j$(nproc)
```

编译完成后，二进制主程序位于 `scripts/` 下（如 `_t-rex-64`），执行 `./t-rex-64 -v` 或使用 `./t-rex-64 -i` 即可启动交互式无状态模式验证。
