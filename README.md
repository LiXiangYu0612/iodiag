# iodiag — Block Device I/O Latency Analyzer

基于 Linux tracefs/blktrace 事件流的块设备 I/O 延迟分析与诊断工具。单二进制,无运行时依赖,支持 x86_64 / aarch64 / ppc64le。

> **源代码不公开。** 本仓库仅提供可执行文件发布与使用文档。
> 问题反馈:minniebaby(lxy4@vastdata.com.cn)

## 下载

到 [Releases](https://github.com/LiXiangYu0612/iodiag/releases/latest) 下载对应架构的二进制(静态编译 + UPX 压缩,上传即用):
`iodiag_x86_64` / `iodiag_aarch64` / `iodiag_ppc64le`

## 功能

- **实时追踪** (`trace`):私有 tracefs instance 采集 block 层 tracepoint,不干扰全局 trace buffer,不依赖系统安装 blktrace
- **离线分析** (`analyze`):直接解析标准 blktrace 二进制文件
- **报告对比** (`compare`):两份 JSON 报告逐指标对比(变更前/后基线)
- **完整 I/O 路径分解**:Q→G(分配)→I(入队)→D(下发)→C(完成),读/写 × 顺序/随机四象限
- **智能诊断**:设备无响应 / completion 丢失 / 调度器饥饿 / 慢 I/O / 延迟毛刺 / 队列深度 / flush-FUA-sync 构成 / 挂载选项等 20+ 类问题,按 ERROR/WARN/INFO 分级并附修复建议
- **延迟热力图**(`-i` 间隔模式)与顶层进程 I/O 排行

## 使用

```bash
# 实时追踪(需要 root;默认 10s,结束窗口含 2s drain 收尾在途 I/O)
sudo ./iodiag_x86_64 trace /dev/sda -d 30
sudo ./iodiag_x86_64 trace /dev/sda,/dev/sdb -d 60   # 多设备
sudo ./iodiag_x86_64 trace /dev/sda -d 60 -i 5       # 每 5s 摘要 + 热力图
sudo ./iodiag_x86_64 trace /dev/sda -d 30 -v         # 逐 I/O 明细

# 离线分析 blktrace 文件(无需 root)
./iodiag_x86_64 analyze sda.blktrace.0 sda.blktrace.1

# 对比两份 JSON 报告(无需 root)
./iodiag_x86_64 compare before.json after.json
```

通用选项:`-d <dur>`、`-i <sec>`、`-v`、`-level all|warn|error`、`-format text|json`。
报告同时输出到 stdout 和二进制所在目录的 `iodiag_output/`。

## 权限要求

| 子命令 | root | 原因 |
|---|---|---|
| `trace` | **必须** | 需要写 tracefs |
| `analyze` / `compare` | 不需要 | 纯文件解析 |

## 正确性

与 blktrace 官方工具做过四方对账验证(实时路径 / blktrace 文件路径 / 独立解析 / `/proc/diskstats` 差分):I/O 计数精确一致,读延迟 Avg/P50/P99/Max 逐位一致。详见 [CHANGELOG](CHANGELOG.md)。

## 已知限制

- tracefs 文本事件流不含请求 ID,会话匹配基于 `设备:扇区:大小` 启发式;WAL/journal 类同扇区重写负载下的单笔延迟归因存在启发式误差(计数与吞吐不受影响)
- flush(零扇区)completion 无法按扇区归属,计入孤儿会话
