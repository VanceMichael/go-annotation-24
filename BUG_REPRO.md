# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

转化得率算出来全部超过 1，三个批次都被判成不合理，可是质量平衡明明是闭合的。

```
$ ./oilctl convert compute
...
    {
      "batch_id": "B-003",
      "feed_mass_kg": 3100,
      "product_volume_l": 3253,
      "product_mass_kg": 2865.893,
      "ratio": 1.0494,
      "balance_gap_kg": 1.007,
      "balanced": true,
      "plausible": false
    }
  ],
  "summary": {
    "batches": 3,
    "feed_mass_kg": 25500,
    "product_mass_kg": 23579.965,
    "mean_ratio": 0.9247,
    "balanced": 3,
    "implausible": 3
  },
  "implausible": 3,
  "unbalanced": 0
}
```

很矛盾的一组数字：

- 每个批次的 `balanced` 都是 true，`unbalanced` 是 0 —— 质量平衡没问题。
- `summary.mean_ratio` 是 0.9247，落在正常区间里，看着是对的。
- 但每个批次自己的 `ratio` 都是 1.04 以上，于是 `plausible` 全是 false、`implausible` 是 3。

也就是说汇总层的平均得率和单批次得率对不上，一个正常一个不正常。得率按项目口径是「产出质量 ÷ 投料质量」，B-003 投料 3100kg、产出质量 2865.893kg，手算是 0.9245，系统给的是 1.0494。

`GET /api/batches/B-003/yield` 返回的 `ratio` 也是 1.0494。

顺带一个可能有关的观察：算出来的 `ratio` 和手算值的比大约是 1.135，三个批次都差不多这个倍数。

请先不要修改代码。先帮我定位根因，讲清楚为什么单批次得率会被整体放大、为什么质量平衡和汇总平均得率反而是对的，并给出实际执行过的复现命令与观察到的输出作为证据。结论确认后再讨论怎么改。

## 含 Bug 版本

- 仓库：VanceMichael/go-annotation-24
- 仓库地址：https://github.com/VanceMichael/go-annotation-24.git
- parent SHA：12f5429f08e98b0d7eac1172b0b045d6ccceb685

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-annotation-24.git bug-repro
cd bug-repro
git checkout --detach 12f5429f08e98b0d7eac1172b0b045d6ccceb685
go test ./internal/convert/ ./internal/report/ -run "TestYieldUsesDensity|TestYieldRatioNotComputedFromVolume|TestYieldAcrossDensities|TestAggregateWeightedMeanRatio|TestConversionRatiosPlausible" -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/convert/ ./internal/report/ -run "TestYieldUsesDensity|TestYieldRatioNotComputedFromVolume|TestYieldAcrossDensities|TestAggregateWeightedMeanRatio|TestConversionRatiosPlausible" -count=1
--- FAIL: TestYieldUsesDensity (0.00s)
    convert_test.go:57: 得率 = 1.0500, 期望约 0.9251（按质量口径）
--- FAIL: TestYieldRatioNotComputedFromVolume (0.00s)
    convert_test.go:73: 得率 = 1.0500 与体积比 1.0500 接近, 说明未做密度换算
--- FAIL: TestYieldAcrossDensities (0.00s)
    convert_test.go:122: 密度 0.870 得率 1.0500 应高于更低密度的 1.0500
--- FAIL: TestAggregateWeightedMeanRatio (0.00s)
    convert_test.go:181: 异常得率批次数 = 2, 期望 0
FAIL
FAIL	wasteoil/internal/convert	0.032s
--- FAIL: TestConversionRatiosPlausible (0.00s)
    report_test.go:109: 异常得率批次数 = 3, 期望 0
FAIL
FAIL	wasteoil/internal/report	0.034s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/convert/ ./internal/report/ -run "TestYieldUsesDensity|TestYieldRatioNotComputedFromVolume|TestYieldAcrossDensities|TestAggregateWeightedMeanRatio|TestConversionRatiosPlausible" -count=1
--- FAIL: TestYieldUsesDensity (0.00s)
    convert_test.go:57: 得率 = 1.0500, 期望约 0.9251（按质量口径）
--- FAIL: TestYieldRatioNotComputedFromVolume (0.00s)
    convert_test.go:73: 得率 = 1.0500 与体积比 1.0500 接近, 说明未做密度换算
--- FAIL: TestYieldAcrossDensities (0.00s)
    convert_test.go:122: 密度 0.870 得率 1.0500 应高于更低密度的 1.0500
--- FAIL: TestAggregateWeightedMeanRatio (0.00s)
    convert_test.go:181: 异常得率批次数 = 2, 期望 0
FAIL
FAIL	wasteoil/internal/convert	0.002s
--- FAIL: TestConversionRatiosPlausible (0.00s)
    report_test.go:109: 异常得率批次数 = 3, 期望 0
FAIL
FAIL	wasteoil/internal/report	0.002s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

目标仓库零改动（git status 干净，无新增、修改或删除文件）。
准确指出出问题的 Go 文件与具体符号。
说明该符号计算得率时取用的数值量纲为什么与得率定义不一致，并解释这一点如何使比值被整体放大约 1/密度 倍、从而越过合理区间上界，导致全部批次被标记为不合理。
解释为什么质量平衡校验与汇总平均得率不受影响，从而说明偏差被局限在单批次得率这一个字段上。
给出实际执行过的复现命令与观察到的输出作为证据，而非仅凭阅读代码推断。
