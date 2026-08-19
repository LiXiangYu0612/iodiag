# Changelog

## v1.1.1 (2026-08-19)

正确性里程碑:与 blktrace 四方对账(tracefs 路径 / blktrace 文件路径 / python 独立解析 / diskstats),I/O 计数与读延迟逐位一致。

- **修复时间戳解析方向错误**:原代码取 header 最后一个冒号**之后**的文本作为时间戳(实际在冒号之前),导致解析永远失败、所有 tracefs 事件静默回退到 `time.Now()`(消费时刻)。Q/C 通常同批消费掩盖了该问题,但延迟统计含消费抖动(实测 avg +13%、P99 +47%)
- tracefs instance 缓冲区提升至 8MB/CPU(默认 ~1.4MB 在突发负载下会静默丢事件);退出时检查 per-CPU overrun 计数,丢事件时明确告警
- drain 窗口内事件发送由 select(随机丢弃)改为阻塞发送
- 新增 `IODIAG_DEBUG_EVENTS` 环境变量:dump 解析结果 + 原始行,用于采集链路对账

## v1.1.0 (2026-08-19)

- **修复读 I/O 在报告中消失**:payload 解析的 "+" 锚点写死 `i>=4`,丢弃了 `block_bio_queue`/`block_getrq` 的 5 字段格式事件;且 `block_rq_issue` 读的 bytes 字段为 0,导致会话 key 断裂。改为锚点位置自适应 + bytes=0 回退 nsectors×512
- **修复 trace 结束误报 "Device unresponsive"**:增加 2s drain 收尾在途 I/O;未完成 I/O 按"设备是否仍在完成其他 I/O"三级分类(ERROR 设备挂死 / WARN completion 未观测 / INFO 在途),不再一律报设备故障
- 改用私有 tracefs instance(`instances/iodiag_<pid>`),不再读写全局 buffer(此前会与其他 ftrace 使用者互吞事件、并在退出时关闭他人已启用的 tracepoint)
- 补启用 `block_bio_backmerge/frontmerge` tracepoint(此前 trace 模式 MergeRate 恒为 0);"低合并率"提示增加顺序占比与并发度双门槛
- 修复同扇区重写(WAL/journal)导致的世代混淆:幻影 40s "Write timeout" 消除,同 key 重排队时丢弃旧 pending
- completion 匹配增加扇区区间覆盖兜底(合并完成的 size 为各 bio 之和)
- 采集改为非阻塞 fd + select 事件驱动:空闲设备上 `-d` 到期 / Ctrl-C 可确定性退出(此前可能无限阻塞)
- trace 模式增加 root 预检,非 root 给出明确提示

## v1.0.0 (2026-06-08)

初始版本:trace/analyze/compare 三模式,tracefs 采集,blktrace 文件解析,I/O 路径五段延迟分解,20+ 类问题诊断,延迟热力图,JSON 报告与对比。
