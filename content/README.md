# 拔刀斩（代号：幻想导数）
🚧 *开发中 | In Development*

基于 UE5 的快节奏砍杀动作游戏 Demo，以"拔刀斩"为核心机制，支持俯视角 (Top-down) 与越肩视角 (Over-the-shoulder) 双视角实时切换。

![](file-20260228025102281.mp4)

## 核心玩法

- 拔刀斩三阶段：蓄力 → 斩击 → 收刀
- 能量机制：收刀回蓝，首斩强化
- 双视角战斗：俯视角（割草，爽，水果忍者精神续作）/ 越肩（博弈，魂，FPS但枪是角色）

## 技术实现

### TD↔TP 视角切换

![](file-20260228025102305.mp4)

通过逆向几何实时计算Camera Offset，在BlendNode外层进行覆盖旋转值，使用`OnBlendResults` 中的二阶Blend结果，消除镜头跳变。

→ [25.12.10-25.12.24 消除视角跳变：基于逆向几何推导与二阶混合修正的 LookAtBlend 算法](25.12.10-25.12.24%20消除视角跳变：基于逆向几何推导与二阶混合修正的%20LookAtBlend%20算法.md)

### 视角状态管理

![](file-20260228025102354.mp4)
拒绝 Subsystem 过度设计，采用 GameplayTag + BlueprintFunctionLibrary 方案，零运行时实例开销实现视角状态同步。

→ [25.12.25 - 26.1.07 拒绝过度设计：为何我放弃 Subsystem 而选用GameplayTag + BlueprintFunctionLibrary管理视角状态](25.12.25%20-%2026.1.07%20拒绝过度设计：为何我放弃%20Subsystem%20而选用GameplayTag%20+%20BlueprintFunctionLibrary管理视角状态.md)

## 开发进度

- [x] Motion Matching
- [x] 拔刀斩 GA
- [x] 视角切换
- [x] 敌人检测与锁定
- [ ] 弹反机制
- [ ] 伤害系统

## 技术栈

UE5 源码 | GAS | Gameplay Camera System | Unreal C++