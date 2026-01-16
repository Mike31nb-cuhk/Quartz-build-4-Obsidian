## 2025.12.20 Bug修复：多Blend同时激活时的跳变问题

### 问题描述

当在SmoothBlend（TD→TP）进行过程中快速触发LookAtBlend（TP→TD）时，画面会在LookAtBlend触发的瞬间出现Rotation跳变。特点：
- 跳变只出现在LookAtBlend被触发的那一瞬间
- 之后blend继续进行时没有跳变
- 位置变化不明显，主要是Rotation的sudden change

### 根因分析

通过阅读`TransientBlendStackCameraNode.cpp`源码，发现GCS的BlendStack处理机制：

```cpp
// InternalPostBlendExecute中按顺序处理每个Entry
for (FResolvedEntry& ResolvedEntry : ResolvedEntries)
{
    EntryBlendEvaluator->BlendResults(BlendParams, BlendResult);
}
·```

当多个Blend同时激活时，它们的`BlendResults`是按顺序执行的，`OutResult`会被逐步修改。

**问题根源**：`OnInitialize`中使用`Params.LastActiveCameraRigInfo.LastResult->CameraPose`（LastPose）计算的`InitOffset`，和`OnBlendResults`第一帧时`OutResult.BlendedResult.CameraPose`（CurPos）不一致。

#### 时序分析

```
帧 N: SmoothBlend正在进行 (TD → TP), 进度50%
       此时触发LookAtBlend (TP → TD)
       
       OnInitialize执行:
       └─ LastPose = TD rig的CameraPose (某个特定rig的固定状态)
       └─ InitOffset 基于 LastPose 计算
       
帧 N+1: OnBlendResults执行:
       └─ OutResult.BlendedResult 已被SmoothBlend处理，是混合后的中间状态！
       └─ CurPos ≠ LastPose.GetLocation()  ← 不一致！
       └─ DispVec方向错误 → Rotation跳变！
```

#### 数学推导

```cpp
// OnInitialize计算的InitOffset:
InitOffset = LastPose.GetLocation() + LastPose.GetRotation().Vector() * len - RootLocation

// OnBlendResults第一帧 (BlendTimer.bDoOnce返回alpha=0):
BasePos = RootLocation + InitOffset 
        = LastPose.GetLocation() + LastPose.GetRotation().Vector() * len

// Rotation计算:
DispVec = BasePos - CurPos
CacheRot = DispVec.ToOrientationRotator()
```

设计假设是`CurPos ≈ LastPose.GetLocation()`，这样`DispVec`的方向会接近`LastPose.GetRotation().Vector()`，Rotation自然和之前一致。

但当SmoothBlend在进行时，`CurPos`是混合后的中间位置，打破了这个假设，导致`DispVec`方向偏离，Rotation跳变。

### 解决方案

**核心思路**：将`InitOffset`的计算从`OnInitialize`移到`OnBlendResults`的**第一帧**，使用实际的`OutResult.BlendedResult.CameraPose`来计算。

#### 修改1：FBlendTimer结构

```cpp
struct FBlendTimer
{
    float CurrentTime = 0.0f;
    float BlendTime = 1.0f;
    bool bIsActive = false;
    bool bIsFirstFrame = true;  // 重命名，明确用途
    
    void Start(const float Duration)
    {
        BlendTime = Duration;
        CurrentTime = 0.0f;
        bIsActive = true;
        bIsFirstFrame = true;  // 在Start时重置！
    }
    
    float Update(float DeltaTime, bool& bOutIsFirstFrame)
    {
        bOutIsFirstFrame = bIsFirstFrame;
        if (bIsFirstFrame) 
        {
            bIsFirstFrame = false;
            return 0.0f;  // 第一帧返回0
        }
        
        if (!bIsActive) return 1.0f;

        CurrentTime += DeltaTime;
        float Alpha = FMath::Clamp(CurrentTime / BlendTime, 0.0f, 1.0f);

        if (Alpha >= 1.0f) bIsActive = false;

        return Alpha;
    }
};
```

#### 修改2：OnInitialize

```cpp
void FLookAtBlendCameraNodeEvaluator::OnInitialize(
    const FCameraNodeEvaluatorInitializeParams& Params, 
    FCameraNodeEvaluationResult& OutResult)
{
    // 不再在这里计算InitOffset
    // InitOffset将在OnBlendResults第一帧计算
}
```

#### 修改3：OnBlendResults

```cpp
void FLookAtBlendCameraNodeEvaluator::OnBlendResults(
    const FCameraNodeBlendParams& Params, 
    FCameraNodeBlendResult& OutResult)
{
    if (PositionEvaluator)
    {
        PositionEvaluator->BlendResults(Params, OutResult);
    }
        
    // Update Rotation
    {
        const FCameraPose& ContextOrigin = Params.ChildParams.EvaluationContext->GetInitialResult().CameraPose;
        FVector RootLocation = ContextOrigin.GetLocation();
        
        bool bIsFirstFrame = false;
        const float Alpha = BlendTimer.Update(Params.ChildParams.DeltaTime, bIsFirstFrame);
        
        // 关键修改：在第一帧使用实际的BlendedResult计算InitOffset
        if (bIsFirstFrame)
        {
            // 使用实际的当前相机状态，而不是OnInitialize时缓存的LastPose
            const FVector ActualCamPos = OutResult.BlendedResult.CameraPose.GetLocation();
            const FRotator ActualCamRot = OutResult.BlendedResult.CameraPose.GetRotation();
            
            float len = FVector::Dist(ActualCamPos, RootLocation);
            // 基于实际状态计算InitOffset
            InitOffset = ActualCamPos + ActualCamRot.Vector() * len - RootLocation;
        }
        
        FVector BasePos = RootLocation;
        const FVector CurrentOffset = FMath::Lerp(InitOffset, FVector(0,0,0), Alpha);
        BasePos += CurrentOffset;
        
        const FVector CurPos = OutResult.BlendedResult.CameraPose.GetLocation();
        const FVector DispVec = BasePos - CurPos;
        CacheRot = DispVec.ToOrientationRotator();
    }
    OutResult.BlendedResult.CameraPose.SetRotation(CacheRot);
}
```

### 为什么这个方案能解决问题

| 对比项 | 修改前 | 修改后 |
|--------|--------|--------|
| InitOffset计算时机 | OnInitialize（transition触发时） | OnBlendResults第一帧 |
| 使用的相机状态 | LastActiveCameraRigInfo.LastResult（某个特定rig） | OutResult.BlendedResult（当前帧实际状态） |
| 与CurPos一致性 | ❌ 可能不一致（其他blend在中间修改了） | ✅ 保证一致（同一帧、同一数据源） |

这个修改确保了无论有多少个Blend同时激活，`InitOffset`的计算都基于**实际当前帧的相机状态**，消除了时序不一致导致的跳变问题。

---

## 2025.12.22 深入理解：TransientBlendStackCameraNode::OnRun 执行流程

### 完整调用链

```
UE引擎每帧
    │
    └─→ APlayerCameraManager::UpdateCamera()
            │
            └─→ AGameplayCamerasPlayerCameraManager::DoUpdateCamera()
                    │
                    └─→ FCameraSystemEvaluator::Update()
                            │
                            └─→ RootEvaluator->Run()
                                    │
                                    └─→ FDefaultRootCameraNodeEvaluator::OnRun()
                                            │
                                            ├─→ BaseLayer->Run()
                                            └─→ MainLayer->Run()  ← MainLayer = TransientBlendStack
                                                    │
                                                    └─→ FTransientBlendStackCameraNodeEvaluator::OnRun()
```

### OnRun 的6个步骤

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              FTransientBlendStackCameraNodeEvaluator::OnRun()                   │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤1: ResolveEntries()                                                  │   │
│  │   - 验证所有 Entry 是否有效                                               │   │
│  │   - 解析 weak pointer 到 strong pointer                                  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤2: InternalPreBlendPrepare()                                         │   │
│  │   - 准备变量表                                                            │   │
│  │   - 收集需要 pre-blend 的参数                                             │   │
│  │   ★★★ 调用 FBlendCameraNodeEvaluator::OnRun() ★★★                         │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤3: InternalPreBlendExecute()                                         │   │
│  │   - 混合所有 entry 的输入变量（如相机参数）                                  │   │
│  │   ★★★ 调用 OnBlendParameters() ★★★                                        │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤4: InternalUpdate()                                                  │   │
│  │   - 运行每个 CameraRig 的 RootNode                                        │   │
│  │   - 每个 rig 独立计算出自己的 CameraPose                                   │   │
│  │   - (这里不调用Blend方法，而是调用各Rig的RootNode.Run)                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤5: InternalPostBlendExecute()                                        │   │
│  │   - 按顺序遍历所有 entry                                                  │   │
│  │   - 将每个 entry 的结果混合到 OutResult                                   │   │
│  │   ★★★ 调用 OnBlendResults() ★★★                                          │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 步骤6: OnRunFinished() + InternalRunFinished()                           │   │
│  │   - 清理标志位                                                            │   │
│  │   - 重置 bInputRunThisFrame, bBlendRunThisFrame 等                        │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 各步骤详解

#### 步骤1: ResolveEntries()

```cpp
TArray<FResolvedEntry> ResolvedEntries;
ResolveEntries(Params, ResolvedEntries);
```

**作用**：验证 BlendStack 中的所有 Entry，把 weak pointer 解析成 strong pointer。因为 Entry 中存储的 EvaluationContext 是弱引用，可能在运行时被销毁，需要先检查有效性。

#### 步骤2: InternalPreBlendPrepare() → 调用 OnRun()

```cpp
// 关键代码：
FBlendCameraNodeEvaluator* EntryBlendEvaluator = Entry.RootEvaluator->GetBlendEvaluator();
if (EntryBlendEvaluator)
{
    EntryBlendEvaluator->Run(CurParams, CurResult);  // ← 调用你的OnRun()
}
```

**作用**：
1. 准备变量表：把全局变量、Context变量、Parameter Setter的值写入每个Entry的VariableTable
2. 收集pre-blend参数：遍历所有需要参数更新的节点，调用`UpdateParameters()`
3. 运行Blend的Run：调用 `EntryBlendEvaluator->Run()` → 最终调用你的 `OnRun()`

**典型用途**：更新 blend 时间/进度，比如 SimpleBlend 在这里计算 `BlendFactor`

#### 步骤3: InternalPreBlendExecute() → 调用 OnBlendParameters()

```cpp
// 关键代码：
FBlendCameraNodeEvaluator* EntryBlendEvaluator = Entry.RootEvaluator->GetBlendEvaluator();
if (EntryBlendEvaluator)
{
    EntryBlendEvaluator->BlendParameters(PreBlendParams, PreBlendResult);  // ← 调用你的OnBlendParameters()
}
```

**作用**：
- 混合所有 Entry 的**输入变量**（CameraRig的参数，如FOV、Offset等）
- 结果写入 `PreBlendVariableTable`
- 然后把混合后的变量**回写**给每个Entry，让它们用混合后的参数去运行

#### 步骤4: InternalUpdate() （不调用Blend方法）

```cpp
// 关键代码：
CurResult.CameraPose = OutResult.CameraPose;  // 从OutResult复制初始pose

FCameraNodeEvaluator* RootEvaluator = Entry.RootEvaluator->GetRootEvaluator();
if (RootEvaluator)
{
    RootEvaluator->Run(CurParams, CurResult);  // 运行CameraRig的RootNode
}
```

**作用**：
- 让每个 CameraRig 独立计算出自己的 `CameraPose`
- 调用的是 CameraRig 的 RootNode，不是 Blend

#### 步骤5: InternalPostBlendExecute() → 调用 OnBlendResults()

```cpp
// 关键代码：
for (FResolvedEntry& ResolvedEntry : ResolvedEntries)
{
    FCameraNodeBlendParams BlendParams(CurParams, CurResult);  // CurResult = 这个Entry的计算结果
    FCameraNodeBlendResult BlendResult(OutResult);             // OutResult = 累积的混合结果
    
    FBlendCameraNodeEvaluator* EntryBlendEvaluator = Entry.RootEvaluator->GetBlendEvaluator();
    if (EntryBlendEvaluator)
    {
        EntryBlendEvaluator->BlendResults(BlendParams, BlendResult);  // ← 调用你的OnBlendResults()
    }
}
```

**作用**：
- **按顺序**遍历所有Entry
- 把每个Entry的 `CurResult`混合到 `OutResult`

### 三个方法的调用顺序示例

假设 BlendStack 有3个 Entry：[TP_Idle, TP_Run, LookAtBlend]

```
每帧 OnRun 被调用时：

┌─────────────────────────────────────────────────────────────────┐
│  Entry[0]                Entry[1]                Entry[2]       │
│  (TP_Idle)               (TP_Run)                (LookAtBlend)  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  步骤2: InternalPreBlendPrepare                                  │
│  ├─→ Entry[0].BlendEvaluator->Run()                            │
│  ├─→ Entry[1].BlendEvaluator->Run()                            │
│  └─→ Entry[2].BlendEvaluator->Run()  ← 你的OnRun()             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  步骤3: InternalPreBlendExecute                                  │
│  ├─→ Entry[0].BlendEvaluator->BlendParameters()                │
│  ├─→ Entry[1].BlendEvaluator->BlendParameters()                │
│  └─→ Entry[2].BlendEvaluator->BlendParameters() ← 你的方法      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  步骤4: InternalUpdate                                           │
│  ├─→ Entry[0].RootNode->Run()  （计算TP_Idle的pose）            │
│  ├─→ Entry[1].RootNode->Run()  （计算TP_Run的pose）             │
│  └─→ Entry[2].RootNode->Run()  （计算TD镜头的pose）             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  步骤5: InternalPostBlendExecute                                 │
│  │                                                              │
│  │   OutResult.CameraPose = BaseLayer处理后的初始pose           │
│  │                                                              │
│  ├─→ Entry[0].BlendEvaluator->BlendResults()                   │
│  │   OutResult 现在 = Lerp(初始, TP_Idle, Factor0)              │
│  │                                                              │
│  ├─→ Entry[1].BlendEvaluator->BlendResults()                   │
│  │   OutResult 现在 = Lerp(上面结果, TP_Run, Factor1)           │
│  │                                                              │
│  └─→ Entry[2].BlendEvaluator->BlendResults() ← 你的方法         │
│      OutResult 现在 = Lerp(上面结果, TD镜头, Factor2)           │
│      ★ 这里的OutResult.BlendedResult.CameraPose是真正的屏幕pose★│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 总结表格

| 方法 | 调用位置 | 传入的Pose | 典型用途 |
|------|---------|-----------|---------|
| `OnRun()` | InternalPreBlendPrepare | `OutResult.CameraPose` = 上一帧的pose | 更新blend进度/时间 |
| `OnBlendParameters()` | InternalPreBlendExecute | `Params.LastCameraPose` = 上一帧pose | 混合输入变量（FOV、Offset等） |
| `OnBlendResults()` | InternalPostBlendExecute | `OutResult.BlendedResult.CameraPose` = **当前帧累积pose** | 混合最终的CameraPose |

### 关键结论

**为什么必须在 `OnBlendResults()` 的第一帧计算 InitOffset？**

因为只有 `OnBlendResults()` 中的 `OutResult.BlendedResult.CameraPose` 才是**经过之前所有Entry混合后的"真正屏幕pose"**。

- `OnInitialize` 时获取的 pose 是某个特定rig的配置pose，不包含正在进行的blend的中间状态
- `OnRun` 时获取的 pose 是上一帧的pose
- 只有 `OnBlendResults` 时的 `OutResult.BlendedResult.CameraPose` 是当前帧、经过所有之前Entry blend后的实际pose
