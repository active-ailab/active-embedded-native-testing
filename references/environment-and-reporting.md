# 环境与报告口径

## 依赖检查

在 Linux 或 WSL 主机执行：

```bash
uname -a
node --version
gcc --version
gcov --version
```

若测试脚本不依赖 Node.js，可省略 Node.js。使用 Clang 时检查：

```bash
clang --version
llvm-cov --version
llvm-profdata --version
```

必须保证覆盖率读取工具与生成覆盖数据的编译器工具链匹配。GCC 生成的 `.gcno/.gcda` 使用对应版本 gcov；Clang 覆盖率使用 `llvm-profdata` 和 `llvm-cov`，不要混用。

可选依赖：

- `lcov`、`genhtml`：生成 HTML 总览。
- `c++filt`：还原 C++ 符号。
- `addr2line`：定位主机测试崩溃地址。
- GoogleTest：仅在用 `TEST()`/`TEST_F()` 编写 C++ 测试时需要；简单 C harness 可自带 `main()`，不依赖 GoogleTest。

## 工作区 GoogleTest

本工作区已包含 GoogleTest 源码：

```text
3rdparty/googletest-main/googletest/include/gtest/gtest.h
3rdparty/googletest-main/googletest/src/gtest-all.cc
3rdparty/googletest-main/googletest/src/gtest_main.cc
```

`gtest_main.cc` 只提供默认的 `main()` 入口，并不包含 GoogleTest 的主体实现。直接使用源码编译时，应同时编译 `gtest-all.cc`，并加入以下头文件路径：

```bash
-I3rdparty/googletest-main/googletest/include \
-I3rdparty/googletest-main/googletest
```

典型命令如下：

```bash
g++ -std=c++17 -g -O0 -pthread --coverage \
  -I3rdparty/googletest-main/googletest/include \
  -I3rdparty/googletest-main/googletest \
  test_case.cc business_under_test.o \
  3rdparty/googletest-main/googletest/src/gtest-all.cc \
  3rdparty/googletest-main/googletest/src/gtest_main.cc \
  -o unit_test
```

使用自定义 `main()` 时不要再编译 `gtest_main.cc`，否则会产生重复入口。C 业务源码可先用 `gcc` 编译为对象文件，再由 `g++` 与 GoogleTest 链接；跨语言头文件应具备 `extern "C"` 保护。GoogleTest 自身及测试代码产生的覆盖数据不得计入业务源码覆盖率。

本工作区的 `packages/services/sport/test/activity_spor_ar/activity_hr_snapshot_test.js` 使用自带断言和 `main()` 的 C harness，因此当前测试不依赖 `gtest_main.cc`。

## 推荐编译参数

Sanitizer 轮次：

```bash
-std=c11 -g -O0 -Wall -Wextra -Werror \
-fno-omit-frame-pointer -fsanitize=address,undefined
```

Coverage 轮次：

```bash
-std=c11 -g -O0 -Wall -Wextra -Werror --coverage
```

根据项目语言版本、既有告警基线和编译器能力调整参数。不要通过全局关闭告警掩盖 harness 或业务源码问题。

## 宏与生成代码

- 从项目实际 `.config`、defconfig 或构建命令获取宏值。
- 在报告中记录会改变控制流的宏，不要只记录测试脚本中模拟的值。
- 测试 protobuf 等生成代码时，注明生成器版本；若只测试上层调用，避免重新生成并污染仓库。

## 业务覆盖率报告模板

| 对象 | 验证方式 | 行覆盖率 | 分支覆盖率 | 说明 |
|---|---|---:|---:|---|
| 原始 API A | 原源码运行 | x/y | x/y | 覆盖正常和失败路径 |
| 原始 API B | 原源码运行 | x/y | x/y | 某宏关闭导致旧分支不可达 |
| 静态函数 C | 原函数体抽取 | x/y | x/y | 已校验签名和调用点 |
| API D | 静态接线审计 | 不计入 | 不计入 | 需目标机协议验证 |

报告总覆盖率时，同时给出分子和分母。不要只给百分比，也不要将测试脚本、mock 或断言代码加入业务覆盖率分母。

## 结论边界

主机测试适合验证纯计算、状态机、内存生命周期、错误返回和容量边界。以下内容通常仍需目标机验证：

- 实际 FATFS/FAL/Sharedp 行为和掉电一致性。
- RTOS 并发、锁、任务切换和实时性。
- 传感器、蓝牙/GDSP 协议及 APP 消费结果。
- 链接脚本、目标架构对齐和目标编译器特有行为。
