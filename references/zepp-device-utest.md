# Zepp Device UTEST

## 使用条件

先确认以下事实，不因目录存在就认定框架可用：

1. `components/utest/utest.c` 和 `utest_shell.c` 参与当前固件编译。
2. `build/.config` 未启用 `NDEBUG`，或项目有等价的调试构建配置。
3. map 中存在 `cmd_utest`、`.utest` 和 `.suite`，目标 shell 已注册 `utest` 命令。
4. 业务模块显式依赖 `components_utest`，测试源码由专用宏控制。

Zepp 当前框架通过链接段自动注册用例，每个用例支持 `init/exec/exit`、独立线程栈、循环、超时和结果统计。

## 用例注册

```c
UTEST_SUITE_EXPORT(example_suite);

UTEST2_EXPORT(example_real_flow,
              5000,
              4096,
              example_init,
              example_exec,
              example_exit,
              UTEST_SUITE_HANDLE(example_suite),
              "packages.services.example",
              "example,device,real_flow");
```

测试宏关闭时，测试源文件、业务薄封装和 `components_utest` 依赖都应消失。不要只屏蔽用例注册而把故障注入留在生产代码中。

## 设备命令

```sh
utest --list-module
utest example_real_flow
utest --suite example_suite
utest --path packages.services.example
utest --tag stress
utest example_stress loops=10 cycles=100 hours=48
```

参数通过以下接口读取：

```c
const char *utest_get_param(const char *key);
int32_t utest_get_param_int(const char *key, int32_t default_value);
```

参数必须检查上下限，避免错误命令造成长时间占用、内存耗尽或 Flash 过度写入。

## 用例分层

### 功能用例

- 调用真实业务 API 或测试宏下的最小薄封装。
- 按生产顺序执行 START、数据输入、PAUSE、RESUME、STOP 等状态。
- 断言业务输出、持久化结果和资源清理，而不是只断言调用返回成功。

### 压力用例

- 覆盖最长时长、最大条目数、环形缓存回绕、反复申请释放和重复生命周期。
- 循环次数和时长可配置，但必须设置设备安全上限。
- 记录执行时长、失败轮次、剩余内存和资源恢复情况；项目没有可靠内存查询接口时应明确说明。

### 容量与故障用例

- 记录数量满场可使用合成 manager/slot 数据验证选择与回收策略，避免删除真实记录。
- 单条记录写满只有在业务存在明确上限或可控写失败注入时才建立用例；不得把总分区上限误当作单条上限。
- 需要验证真实 ENOSPC、目录删除或同步恢复时，使用隔离分区或专用测试设备，并在执行前确认清理和恢复方案。

## 能力边界

- 设备 UTEST 不内建 mock、ASan、UBSan 或 gcov。
- 静态函数不能从独立测试文件直接调用；优先抽取为内部模块，其次使用测试宏保护的薄封装。
- 超时会终止测试线程，可能无法自动释放业务锁或恢复外设状态。
- UTEST 通过只证明断言覆盖的目标机行为，不代表 APP 同步、展示或完整协议链路已经验证。

因此测试报告应把主机 sanitizer/coverage、设备 UTEST 和端到端实测分开列出。
