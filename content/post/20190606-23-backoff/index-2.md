---
title: TiDB - 2PC & Backoff
date: 2019-06-05T00:00:00+08:00
lastmod: 2019-06-06T00:00:00+08:00
categories:
  - TiDB
tags:
  - AirPlan
  - 2PC
  - backoff
---
## 0x00 源

TiDB to TiKV 之间的网络状态(图片来自与 PingCAP 官网[锁冲突](https://docs.pingcap.com/zh/tidb/stable/troubleshoot-lock-conflicts))「该图包含悲观锁逻辑，来自 2020 年的 PingCAP 官网」

![TiDB to TiKV](./troubleshooting-lock-pic-05.png)

## 0x01 2PC

2PC (Two-phaseCommit) 是指在计算机网络以及数据库领域内，为了使基于分布式系统架构下的所有节点在进行事务提交时保持一致性而设计的一种算法 (Algorithm)。第一阶段：准备阶段(投票阶段)和第二阶段：提交阶段（执行阶段）。

在分布式系统中，每个节点虽然可以知晓自己的操作时成功或者失败，却无法知道其他节点的操作的成功或失败。

当一个事务跨越多个节点时，为了保持事务的 ACID 特性，需要引入一个作为协调者的组件来统一掌控所有节点(称作参与者) 的操作结果，并最终指示这些节点是否要把操作结果进行真正的提交(比如将更新后的数据写入磁盘等等)，参与者将操作成败通知协调者，再由协调者根据所有参与者的反馈情报决定各参与者是否要提交操作还是中止操作。

> see store/tikv/2pc.go // 内容过于复杂，没有理解
> 2PC 模型

![2pc (Two-phaseCommit)](./2pc.webp)

> 乐观锁 2PC in TiDB

![TiDB-2PC](./troubleshooting-lock-pic-01.png)  

> 悲观锁 2PC in TiDB

![TiDB-2PC-Pessimistic-lock](./troubleshooting-lock-pic-06.png)  

```golang
const bytesPerMiB = 1024 * 1024

func txnLockTTL(startTime time.Time, txnSize int) uint64 {
	// Increase lockTTL for large transactions.
	// The formula is `ttl = ttlFactor * sqrt(sizeInMiB)`.
	// When writeSize is less than 256KB, the base ttl is defaultTTL (3s);
	// When writeSize is 1MiB, 4MiB, or 10MiB, ttl is 6s, 12s, 20s correspondingly;
	lockTTL := defaultLockTTL
	if txnSize >= txnCommitBatchSize {
		sizeMiB := float64(txnSize) / bytesPerMiB
		lockTTL = uint64(float64(ttlFactor) * math.Sqrt(sizeMiB))
		if lockTTL < defaultLockTTL {
			lockTTL = defaultLockTTL
		}
		if lockTTL > ManagedLockTTL {
			lockTTL = ManagedLockTTL
		}
	}

	// Increase lockTTL by the transaction's read time.
	// When resolving a lock, we compare current ts and startTS+lockTTL to decide whether to clean up. If a txn
	// takes a long time to read, increasing its TTL will help to prevent it from been aborted soon after prewrite.
	elapsed := time.Since(startTime) / time.Millisecond
	return lockTTL + uint64(elapsed)
}
```

## 0x02 RPC

RPC（Remote Procedure Call Protocol 远程过程调用）是分布式架构的核心之一  

- TiDB 2.0 之后使用的 Google RPC 协议 [gRPC](https://godoc.org/google.golang.org/grpc)
  - 同步调用：客户端调用服务方方法，等待直到服务方返回结果或者超时，再继续自己的操作
  - 异步调用：客户端把消息发送给中间件，不再等待服务端返回，直接继续自己的操作

![Remote Procedure Call Protocol](./rpc.jpg)

## 0x03 Backoff

网络之间建立链接会产生各种各样的状态，只做重连机制会造成网络拥堵或者网络风暴，backoff 机制降低资源开销「CPU、时间」，提高网络资源再次链接成功率。

- RPC 网络交互之间需要一个 backoff 机制 [gRPC backoff](https://godoc.org/google.golang.org/grpc/backoff)
  - Backoff 文档解释 [gRPC backoff protocol](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)
  - Backoff Full jitter、Equal jitter、Decorrelated jitter、no jitter 性能测试 [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

```go
const (
	// NoJitter makes the backoff sequence strict exponential.
	NoJitter = 1 + iota
	// FullJitter applies random factors to strict exponential.
	FullJitter
	// EqualJitter is also randomized, but prevents very short sleeps.
	EqualJitter
	// DecorrJitter increases the maximum jitter based on the last random value.
	DecorrJitter
)

// 定义 metrics 信息
var (
	tikvBackoffHistogramRPC        = metrics.TiKVBackoffHistogram.WithLabelValues("tikvRPC")
	tikvBackoffHistogramLock       = metrics.TiKVBackoffHistogram.WithLabelValues("txnLock")
	tikvBackoffHistogramLockFast   = metrics.TiKVBackoffHistogram.WithLabelValues("tikvLockFast")
	tikvBackoffHistogramPD         = metrics.TiKVBackoffHistogram.WithLabelValues("	")
	tikvBackoffHistogramRegionMiss = metrics.TiKVBackoffHistogram.WithLabelValues("regionMiss")
	tikvBackoffHistogramServerBusy = metrics.TiKVBackoffHistogram.WithLabelValues("serverBusy")
	tikvBackoffHistogramStaleCmd   = metrics.TiKVBackoffHistogram.WithLabelValues("staleCommand")
	tikvBackoffHistogramEmpty      = metrics.TiKVBackoffHistogram.WithLabelValues("")
)

// 定义 backoff 回避机制
// 重试间隔定义、最大重试时间定义、backoff jitter 定义
func NewBackoffFn(base, cap, jitter int) func(ctx context.Context, maxSleepMs int) int {
	if base < 2 {
		// Top prevent panic in 'rand.Intn'.
		base = 2
	}
	attempts := 0
	lastSleep := base
	return func(ctx context.Context, maxSleepMs int) int {
		var sleep int
		switch jitter {
		case NoJitter:
			sleep = expo(base, cap, attempts)
		case FullJitter:
			v := expo(base, cap, attempts)
			sleep = int(fastrand.Uint32N(uint32(v)))
		case EqualJitter:
			v := expo(base, cap, attempts)
			sleep = v/2 + int(fastrand.Uint32N(uint32(v/2)))
		case DecorrJitter:
			sleep = int(math.Min(float64(cap), float64(base+int(fastrand.Uint32N(uint32(lastSleep*3-base))))))
		}
		logutil.BgLogger().Debug("backoff",
			zap.Int("base", base),
			zap.Int("sleep", sleep))

		realSleep := sleep
		// when set maxSleepMs >= 0 in `tikv.BackoffWithMaxSleep` will force sleep maxSleepMs milliseconds.
		if maxSleepMs >= 0 && realSleep > maxSleepMs {
			realSleep = maxSleepMs
		}
		select {
		case <-time.After(time.Duration(realSleep) * time.Millisecond):
			attempts++
			lastSleep = sleep
			return realSleep
		case <-ctx.Done():
			return 0
		}
	}
}

// 对应 func NewBackoffFn(base, cap, jitter int) 函数返回信息
func (t backoffType) createFn(vars *kv.Variables) func(context.Context, int) int {
	if vars.Hook != nil {
		vars.Hook(t.String(), vars)
	}
	switch t {
	case boTiKVRPC:
		return NewBackoffFn(100, 2000, EqualJitter)
	case BoTxnLock:
		return NewBackoffFn(200, 3000, EqualJitter)
	case boTxnLockFast:
		return NewBackoffFn(vars.BackoffLockFast, 3000, EqualJitter)
	case BoPDRPC:
		return NewBackoffFn(500, 3000, EqualJitter)
	case BoRegionMiss:
		// change base time to 2ms, because it may recover soon.
		return NewBackoffFn(2, 500, NoJitter)
	case boTxnNotFound:
		return NewBackoffFn(2, 500, NoJitter)
	case boServerBusy:
		return NewBackoffFn(2000, 10000, EqualJitter)
	case boStaleCmd:
		return NewBackoffFn(2, 1000, NoJitter)
	}
	return nil
}
```

## 0x04 Resolve lock

> see store/tikv/lock_resolver.go

对应监控图 tidb dashboard -- kv error 中的 resolve lock 部分 `sum(rate(tidb_tikvclient_lock_resolver_actions_total[1m])) by (type)`

```go

var (
	tikvLockResolverCountWithBatchResolve             = metrics.TiKVLockResolverCounter.WithLabelValues("batch_resolve")
	tikvLockResolverCountWithExpired                  = metrics.TiKVLockResolverCounter.WithLabelValues("expired")
	tikvLockResolverCountWithNotExpired               = metrics.TiKVLockResolverCounter.WithLabelValues("not_expired")
	tikvLockResolverCountWithWaitExpired              = metrics.TiKVLockResolverCounter.WithLabelValues("wait_expired")
	tikvLockResolverCountWithResolve                  = metrics.TiKVLockResolverCounter.WithLabelValues("resolve")
	tikvLockResolverCountWithResolveForWrite          = metrics.TiKVLockResolverCounter.WithLabelValues("resolve_for_write")
	tikvLockResolverCountWithWriteConflict            = metrics.TiKVLockResolverCounter.WithLabelValues("write_conflict")
	tikvLockResolverCountWithQueryTxnStatus           = metrics.TiKVLockResolverCounter.WithLabelValues("query_txn_status")
	tikvLockResolverCountWithQueryTxnStatusCommitted  = metrics.TiKVLockResolverCounter.WithLabelValues("query_txn_status_committed")
	tikvLockResolverCountWithQueryTxnStatusRolledBack = metrics.TiKVLockResolverCounter.WithLabelValues("query_txn_status_rolled_back")
	tikvLockResolverCountWithResolveLocks             = metrics.TiKVLockResolverCounter.WithLabelValues("query_resolve_locks")
	tikvLockResolverCountWithResolveLockLite          = metrics.TiKVLockResolverCounter.WithLabelValues("query_resolve_lock_lite")
)

	// CheckTxnStatus may meet the following cases:
	// 1. LOCK
	// 1.1 Lock expired -- orphan lock, fail to update TTL, crash recovery etc.
	// 1.2 Lock TTL -- active transaction holding the lock.
	// 2. NO LOCK
	// 2.1 Txn Committed
	// 2.2 Txn Rollbacked -- rollback itself, rollback by others, GC tomb etc.
  // 2.3 No lock -- pessimistic lock rollback, concurrence prewrite.


query_txn_status


```


## 0x05 Case








