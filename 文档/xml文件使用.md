# Navigation2 行为树 XML 文件的框架与格式详解

## 一、基本结构

Navigation2 中的行为树 XML 文件遵循一个特定的结构，主要由以下部分组成：

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">
    <!-- 主要行为树内容 -->
  </BehaviorTree>
  
  <!-- 可选的其他行为树定义 -->
  <BehaviorTree ID="SubTree1">
    <!-- 子树内容 -->
  </BehaviorTree>
  
  <!-- 可选的自定义节点模型 -->
  <TreeNodeModel>
    <!-- 节点模型定义 -->
  </TreeNodeModel>
</root>
```

### 主要元素解析：

1. **`<root>`**: 最外层的包装元素
   - `main_tree_to_execute` 属性指定默认执行的树的 ID

2. **`<BehaviorTree>`**: 定义一个行为树
   - `ID` 属性给这个树一个唯一标识符
   - 可以定义多个行为树，在同一个文件中

3. **`<TreeNodeModel>`**: 可选的节点模型定义，用于自定义节点

## 二、节点类型及语法

行为树由多种类型的节点组成，每种类型有特定的功能和语法：

### 1. 控制节点

控制节点决定子节点的执行顺序和条件：

```xml
<!-- 序列节点：按顺序执行所有子节点，全部成功才返回成功 -->
<Sequence name="DoSequentially">
  <子节点1 />
  <子节点2 />
</Sequence>

<!-- 选择节点：按顺序尝试子节点，一个成功就返回成功 -->
<Fallback name="TryUntilSuccess">
  <子节点1 />
  <子节点2 />
</Fallback>

<!-- 并行节点：同时执行所有子节点 -->
<Parallel success_threshold="2" failure_threshold="1">
  <子节点1 />
  <子节点2 />
  <子节点3 />
</Parallel>
```

### 2. 动作节点

动作节点执行具体的操作，如路径规划、移动控制等：

```xml
<!-- 路径规划节点 -->
<ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>

<!-- 路径执行节点 -->
<FollowPath path="{path}" controller_id="DWB"/>

<!-- 旋转节点 -->
<Spin spin_dist="1.57"/>
```

### 3. 条件节点

条件节点检查特定条件，不执行操作：

```xml
<!-- 检查目标是否更新 -->
<GoalUpdated/>

<!-- 检查是否到达目标 -->
<GoalReached distance_threshold="0.15"/>

<!-- 检查是否接收到初始位姿 -->
<InitialPoseReceived/>
```

### 4. 装饰器节点

装饰器节点修改单个子节点的行为：

```xml
<!-- 速率控制器：限制子节点执行频率 -->
<RateController hz="1.0">
  <子节点 />
</RateController>

<!-- 重复执行：多次执行子节点 -->
<RetryUntilSuccessful num_attempts="3">
  <子节点 />
</RetryUntilSuccessful>

<!-- 超时控制：限制子节点执行时间 -->
<TimeoutNode timeout_ms="5000">
  <子节点 />
</TimeoutNode>
```

### 5. 子树引用

可以引用在同一文件或其他文件中定义的行为树：

```xml
<!-- 引用同一文件中的子树 -->
<SubTree ID="RecoverySubTree" />

<!-- 带参数的子树引用 -->
<SubTree ID="NavigationTree" goal="{goal}" path="{path}" />
```

## 三、参数和黑板使用

### 1. 节点参数

节点可以通过属性接收参数：

```xml
<BackUp backup_dist="0.15" backup_speed="0.025"/>
```

### 2. 黑板变量引用

使用 `{}` 语法引用黑板变量：

```xml
<ComputePathToPose goal="{goal}" path="{path}" />
```

### 3. 端口映射

在复杂树中，可以将父节点的端口映射到子节点：

```xml
<Sequence name="NavigateSequence">
  <ComputePathToPose goal="{goal}" path="{path}" />
  <FollowPath path="{path}" />  <!-- 使用上一节点的输出 -->
</Sequence>
```

## 四、特殊节点

Navigation2 包含一些特殊的控制节点，用于导航特定需求：

### 1. 恢复节点

```xml
<RecoveryNode number_of_retries="3" name="RecoverySequence">
  <主要行为 />  <!-- 第一个子节点：主要行为 -->
  <恢复行为 />  <!-- 第二个子节点：恢复行为 -->
</RecoveryNode>
```

当主要行为失败时，执行恢复行为，然后重试主要行为，最多重试指定次数。

### 2. 管道序列

```xml
<PipelineSequence name="NavigateWithReplanning">
  <ComputePathToPose goal="{goal}" path="{path}" />
  <FollowPath path="{path}" />
</PipelineSequence>
```

类似于普通序列，但允许前一个节点完成部分工作后，后一个节点就开始执行。

### 3. 响应式回退

```xml
<ReactiveFallback name="RecoveryFallback">
  <GoalUpdated />  <!-- 条件节点 -->
  <RecoveryActions />  <!-- 只有条件失败才执行 -->
</ReactiveFallback>
```

与普通回退类似，但会在每次执行时重新评估条件节点。

## 五、完整示例分析

下面是一个简化版的单点导航行为树，展示了各种节点的组合使用：

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">
    <RecoveryNode number_of_retries="6" name="NavigateRecovery">
      <!-- 主要导航流程 -->
      <PipelineSequence name="NavigateWithReplanning">
        <!-- 路径规划 -->
        <RateController hz="1.0">
          <RecoveryNode number_of_retries="1" name="ComputePathToPose">
            <ComputePathToPose goal="{goal}" path="{path}" planner_id="GridBased"/>
            <ReactiveFallback name="ComputePathRecovery">
              <GoalUpdated/>
              <ClearEntireCostmap service_name="global_costmap/clear_entirely_global_costmap"/>
            </ReactiveFallback>
          </RecoveryNode>
        </RateController>
        
        <!-- 路径执行 -->
        <RecoveryNode number_of_retries="1" name="FollowPath">
          <FollowPath path="{path}" controller_id="DWB"/>
          <ReactiveFallback name="FollowPathRecovery">
            <GoalUpdated/>
            <ClearEntireCostmap service_name="local_costmap/clear_entirely_local_costmap"/>
          </ReactiveFallback>
        </RecoveryNode>
      </PipelineSequence>
      
      <!-- 主要恢复行为 -->
      <ReactiveFallback name="RecoveryFallback">
        <GoalUpdated/>
        <Round-Robin name="RecoveryActions">
          <Sequence name="ClearingActions">
            <ClearEntireCostmap service_name="local_costmap/clear_entirely_local_costmap"/>
            <ClearEntireCostmap service_name="global_costmap/clear_entirely_global_costmap"/>
          </Sequence>
          <Spin spin_dist="1.57"/>
          <Wait wait_duration="5"/>
          <BackUp backup_dist="0.15" backup_speed="0.025"/>
        </Round-Robin>
      </ReactiveFallback>
    </RecoveryNode>
  </BehaviorTree>
</root>
```

### 分析：

1. **最外层结构**：
   - `<root>` 元素指定主树为 "MainTree"
   - `<BehaviorTree>` 定义了 ID 为 "MainTree" 的行为树

2. **主要控制流**：
   - 最外层的 `<RecoveryNode>` 尝试执行导航流程，失败后执行恢复行为
   - `<PipelineSequence>` 定义了规划和执行的主要序列

3. **路径规划部分**：
   - `<RateController>` 限制规划频率为 1Hz
   - 内部 `<RecoveryNode>` 执行 `<ComputePathToPose>`，失败时尝试清除代价地图

4. **路径执行部分**：
   - 另一个 `<RecoveryNode>` 执行 `<FollowPath>`，失败时尝试清除代价地图

5. **恢复行为部分**：
   - `<ReactiveFallback>` 先检查目标是否更新
   - `<Round-Robin>` 尝试多种恢复行为：清除代价地图、旋转、等待、后退

## 六、树的组织与重用

对于复杂项目，可以组织和重用行为树：

### 1. 多树文件

一个文件中定义多个树，并通过 `<SubTree>` 引用：

```xml
<root main_tree_to_execute="MainTree">
  <BehaviorTree ID="MainTree">
    <Sequence>
      <SubTree ID="PlanningSubTree" />
      <SubTree ID="ExecutionSubTree" />
    </Sequence>
  </BehaviorTree>
  
  <BehaviorTree ID="PlanningSubTree">
    <!-- 规划逻辑 -->
  </BehaviorTree>
  
  <BehaviorTree ID="ExecutionSubTree">
    <!-- 执行逻辑 -->
  </BehaviorTree>
</root>
```

### 2. 跨文件引用

通过 `path` 属性引用其他文件中的树：

```xml
<SubTree ID="RecoveryTree" path="recovery_behaviors.xml" />
```

## 七、总结

Navigation2 行为树 XML 文件的框架和格式具有以下特点：

1. **分层结构**：使用嵌套的节点定义复杂的导航逻辑
2. **多种节点类型**：控制节点、动作节点、条件节点和装饰器节点
3. **参数配置**：通过属性和黑板变量设置节点参数
4. **模块化**：支持子树引用和重用
5. **特殊控制节点**：为导航场景定制的特殊控制节点

这种格式允许以直观的方式描述复杂的导航策略，并支持灵活的配置和修改。理解这种框架有助于创建或修改行为树，以适应不同的导航需求。
