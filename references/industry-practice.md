# 业界实践与方法边界

## 公开实践

主机侧嵌入式 C/C++ 测试是成熟工程方法。以下结论来自项目或厂商官方文档：

| 组织或项目 | 公开能力 | 与本 skill 的对应关系 |
|---|---|---|
| Google Pigweed | 同时支持 host-side 和 on-device unit tests；`pw_unit_test` 可生成主机测试入口，也提供 GoogleTest 后端 | 主机快速回归与设备测试分层；测试入口、断言框架可替换 |
| GoogleTest | Google 按自身跨平台代码测试需求开发的 C++ 测试框架，支持 Linux、Windows 和 macOS | 可组织主机测试并链接由 GCC 编译的 C 业务对象；不是嵌入式硬件模拟器 |
| Arm Mbed OS | 单元测试可由主机原生工具构建为隔离可执行程序；测试体系用于本地开发和 CI，同时保留设备侧测试 | 原始业务对象与测试入口分离；主机单元测试和板端集成测试互补 |
| Zephyr `native_sim` | 将内核、库和应用编译为普通 Linux 可执行文件，支持 ASan、UBSan 和覆盖率 | 证明嵌入式软件可在主机执行 sanitizer 和 coverage，但硬件相关行为仍需设备验证 |

公开资料：

- [Pigweed host tests](https://pigweed.dev/showcases/sense/host_tests.html)
- [Pigweed pw_unit_test](https://pigweed.dev/pw_unit_test/)
- [GoogleTest Primer](https://google.github.io/googletest/primer.html)
- [Arm Mbed OS testing](https://os.mbed.com/docs/mbed-os/v5.15/tools/testing.html)
- [Zephyr native_sim](https://docs.zephyrproject.org/latest/boards/native/native_sim/doc/index.html)

以上资料能够证明相应组织或项目公开维护和支持这些能力。除官方文档明确陈述的内容外，不应进一步推断某家公司所有内部产品都使用同一测试方式。

## 与本 skill 的对应关系

```text
同一份嵌入式业务源码
        |
        +-- 正式固件构建：真实 RTOS、VFS、硬件和协议依赖
        |
        +-- 主机测试构建：mock 时间、内存、存储和系统服务
                              |
                              +-- C harness 或 GoogleTest
                              +-- ASan/UBSan
                              +-- gcov/llvm-cov
```

推荐优先级：

1. 将原始 `.c` 作为独立对象编译，以公开业务 API 驱动测试，链接 mock 依赖。
2. 确实需要验证静态函数时，可在专用白盒测试翻译单元中包含原始 `.c`，但测试目标不得进入固件构建。
3. 完整翻译单元暂时无法解耦时，才抽取未经改写的真实函数体；这属于过渡方案。
4. 不要复制或重写算法后声称覆盖了原始业务代码。

## 适用边界

主机测试适合纯计算、状态机、错误处理、内存生命周期、容量边界和历史缺陷回归。它不能单独证明以下行为正确：

- 目标文件系统的掉电一致性和介质磨损行为。
- RTOS 的真实调度、锁竞争和中断时序。
- 传感器、蓝牙、GDSP 及 APP 端协议消费。
- ARM ABI、链接脚本、目标编译器扩展和硬件寄存器访问。

团队推广时应把主机测试、目标机集成测试和实际业务验收分别记录，不能用主机覆盖率替代后二者。
