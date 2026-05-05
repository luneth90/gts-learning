# GTS World Model — Week 0 交付物

> 目标：先建立三层世界观，不急着学细节。说明白 Geometry / Topology / Semantics 分别关心什么，并给 20 个场景写出三层状态和失败类型。

## 三层定义

| 层 | 关心什么 | 状态保存什么 |
|----|----------|--------------|
| **Geometry** | metric state — 物体在哪里、形状如何、相机如何看它 | 坐标、深度、点云、位姿、占据网格 |
| **Topology** | structural relation — 谁接触谁、谁支撑谁、空间是否连通 | contact、support、inside、occlusion、reachable、Betti 数、事件序列 |
| **Semantics** | task meaning — "放进去"对应哪个拓扑目标、成功条件是什么 | goal predicate、subgoals、plan、success check、failure explanation |

### 三者关系

```
RGB-D / scene
  → GeometryState（度量世界）
  → TopologyState（结构压缩）
  → SemanticState（任务落地）
```

**核心原则：**
- Geometry 变但 Topology 不变 ← 拓扑是稳定结构
- Topology 不变但 Semantics 可能变 ← 任务目标不同
- 失败可以定位到三层中的某一层

## 20 个场景分类表

---

### 场景 1：杯子在桌上

```python
Geometry = {
    "cup_centroid":     (0.0, 0.15, 0.85),  # 杯体质心（杯高约 0.1，半径约 0.04）
    "table_plane":      (0, 0, 1, -0.8),    # 平面方程 0x+0y+1z-0.8=0，即桌面高度 z=0.8
    "cup_bottom_z":     0.80,                # 杯底高度（= 桌面高度，坐在桌面上）
}
Topology = {
    "support(cup, table)": True,             # ← 支撑成立
}
Semantics = {
    "instruction":  "put the cup on the table",
    "goal":         "support(cup, table) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| cup 坐标 Z 写错了（飘在空中）| Geometry | 度量错误 |
| cup 在桌面平移 10cm，支撑关系不变 | Topology | 虽然 Geometry 变了，但 `support` 仍成立 |
| cup 在桌上但任务是"放进碗里" | Semantics | 拓扑正确但 goal 错了 |

---

### 场景 2：苹果在碗里

```python
Geometry = {
    "apple_centroid":   (0.2, 0.15, 0.05),
    "bowl_centroid":    (0.18, 0.14, 0.0),
    "bowl_rim_radius":  0.06,
    "apple_radius":     0.02,
    "apple_to_bowl_base": 0.03,
}
Topology = {
    "inside(apple, bowl)":  True,            # containment 关系
    "support(bowl, table)": True,
}
Semantics = {
    "instruction":  "put the apple in the bowl",
    "goal":         "inside(apple, bowl) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| apple 坐标飘到了碗外 | Geometry | 度量错了，inside 跟着错 |
| apple 放在碗旁边紧贴着，距离只有 0.001 | Topology | 几何"很近"但 inside=False，near≠inside |
| apple 在碗里，但任务说"放在桌上" | Semantics | 拓扑正确但 goal 不对 |

---

### 场景 3：抽屉开了一半

```python
Geometry = {
    "drawer_joint_angle": 0.5,              # 开到一半
    "drawer_linear_displacement": 0.15,     # 滑出 15cm
}
Topology = {
    "articulation_state(drawer)": "open_partial",
    "inside(desk, drawer)": True,            # 抽屉仍在桌体内
}
Semantics = {
    "instruction":  "open the drawer",
    "goal":         "articulation_state(drawer) == open",
    "status":       "in_progress",           # ← 部分完成，不等于完成
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| joint angle 读错了（传感器噪声） | Geometry | 度量值不对 |
| 关节角 0.5，算 open_partial 没问题，但它实际卡住了 | Topology | 静态判断对，但缺失了"物理约束"信息 |
| 认为 open_partial 就是完成 | Semantics | goal 没区分 partial 和 full |

---

### 场景 4：机器人碰到方块

```python
Geometry = {
    "gripper_centroid":     (0.5, 0.3, 0.4),
    "cube_centroid":        (0.5, 0.3, 0.38),  # cube 在 gripper 下方
    "gripper_to_cube_dist": 0.02,                # 即将接触
    "contact_threshold":    0.005,
}
Topology = {
    "contact(gripper, cube)": True,              # 距离 < threshold
}
Semantics = {
    "instruction":  "grasp the cube",
    "goal":         "contact(gripper, cube) == True",
    "status":       "grasp_begin",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| gripper 坐标漂移，实际没碰到 | Geometry | 视觉估计的位姿不准 |
| contact 判了 True，但 gripper 是侧面靠近、无力学接触 | Topology | 只看距离、没考虑力/法向 |
| contact 成立就以为 grasp 成功，其实还没用力 | Semantics | contact ≠ grip force |

---

### 场景 5：障碍物挡住通路

```python
Geometry = {
    "start":    (0, 0, 0),
    "goal":     (1, 0, 0),
    "obstacle": (0.5, 0, 0.4),
    "occupancy_grid": [[...]],   # 0.5 处被占据
}
Topology = {
    "free_space_components": 2,              # 障碍物把空间切成两块
    "reachable(start, goal)": False,         # 不连通
}
Semantics = {
    "instruction":  "go to the other side",
    "goal":         "reachable(start, goal) == True",
    "status":       "blocked",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| obstacle 位置没检测到 | Geometry | occupancy 少了一个障碍 |
| occupancy 分辨率不够，障碍物"漏"了一个缝 | Geometry → Topology | 度量不够精细，导致拓扑误判 |
| reachable=False 但没判断"能不能绕过去" | Semantics | 没考虑路径存在但需要改方向 |

---

### 场景 6：杯子在桌面平移

```python
Geometry = {
    "cup_centroid_0": (0.0, 0.15, 0.8),      # 平移前
    "cup_centroid_1": (0.3, 0.15, 0.8),      # 平移后（X 变了 30cm）
    "delta":          0.3,                     # 几何变化明显
}
Topology = {
    "support(cup, table)": True,              # 平移前
    "support(cup, table)": True,              # 平移后 → 没变！
}
Semantics = {
    "instruction":  "it's still on the table",
    "goal":         "support(cup, table) == True",
    "status":       "unchanged",              # ← geometry 变、topology 不变
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| — | — | 这个场景本身没有失败。它是用来理解"几何变了但拓扑没变"的。 |

---

### 场景 7：杯子从桌上掉下

```python
Geometry = {
    "cup_centroid_0": (0.0, 0.15, 0.8),      # 掉落前：在桌上
    "cup_centroid_1": (0.0, 0.15, 0.2),      # 掉落后：在地面
    "velocity_z":      -1.5,                   # 下落速度
}
Topology = {
    "support(cup, table)": True,              # 掉落前
    "support(cup, table)": False,             # 掉落后 —— 关系断裂！
    "contact(cup, floor)":  True,             # 新关系产生
}
Semantics = {
    "instruction":  "keep the cup on the table",
    "goal":         "support(cup, table) == True",
    "status":       "failure",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| Z 坐标错但支撑判 True（cup 其实已掉） | Geometry → Topology | 度量错了导致拓扑误判 |
| support 断了但认为是"平移" | Topology | 没检测到 support_break 事件 |
| cup 掉了但系统还是认为任务完成 | Semantics | 没做 success check |

---

### 场景 8：苹果靠近碗

```python
Geometry = {
    "apple_centroid": (0.22, 0.15, 0.12),
    "bowl_centroid":  (0.18, 0.14, 0.0),
    "distance":       0.052,                   # 距离很近
}
Topology = {
    "inside(apple, bowl)":  False,            # ← 近 ≠ 在里面
    "near(apple, bowl)":    True,             # 这是一个 metric relation，不是拓扑
}
Semantics = {
    "instruction":  "put the apple in the bowl",
    "goal":         "inside(apple, bowl) == True",
    "status":       "not_ yet",               # ← near success ≠ real success
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 距离错判为 "inside" | Topology | 把 metric relation (near) 当成了 topological relation (inside) |
| apple 离碗只有 1mm 但不在碗内 | Topology | 几何"极近"不能替代 containment 检测 |
| 认为任务已经完成（near success 幻觉） | Semantics | success check 太宽松 |

---

### 场景 9：门开一条缝

```python
Geometry = {
    "door_angle": 0.05,                       # 只开了 5 度
    "door_width":  0.8,
    "gap_size":    0.07,                       # 缝宽约 7cm
}
Topology = {
    "reachable(start, goal)": False,          # 缝太小，仍不可通过
    "free_space_components": 2,               # 连通性还没变
}
Semantics = {
    "instruction":  "go through the door",
    "goal":         "reachable(start, goal) == True",
    "status":       "blocked",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| — | — | 核心教训：Geometry 变了（door angle 从 0→0.05），但 Topology 没变（free-space 仍不连通）。几何变化未必改变拓扑结构。 |

---

### 场景 10：门完全打开

```python
Geometry = {
    "door_angle": 1.57,                       # 开了 90 度
    "gap_size":   0.8,
}
Topology = {
    "reachable(start, goal)": True,           # 可通过！
    "free_space_components": 1,               # 连通了
}
Semantics = {
    "instruction":  "go through the door",
    "goal":         "reachable(start, goal) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| — | — | 门开 90 度 → free-space 连通。对比场景 9：同一条缝，只是角度不同，拓扑完全不同。 |

---

### 场景 11：方块叠在方块上

```python
Geometry = {
    "block_top_centroid":        (0, 0, 0.2),
    "block_bottom_centroid":     (0, 0, 0.1),
    "block_top_bottom_z":        0.095,       # 上方方块底面
    "block_bottom_top_z":        0.095,       # 下方方块顶面
    "contact_normal":            (0, 0, -1),  # 法向朝下=重力方向
}
Topology = {
    "support(block_top, block_bottom)": True, # 支撑方向正确
    "stack_height": 2,
}
Semantics = {
    "instruction":  "stack the blocks",
    "goal":         "support(block_top, block_bottom) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 上方方块歪了，但 centroid 在下方方块上方 | Geometry | 只看 center 不够，要检查 contact plane |
| 判定 support 只看 Z 高度近似 | Topology | support 方向错了（比如侧翻）仍判 True |
| 判定 support 只看高度，没考虑 static stability | Topology | 几何支撑和力学支撑是两回事 |

---

### 场景 12：方块悬空

```python
Geometry = {
    "block_centroid": (0, 0, 0.5),
    "block_bottom_z": 0.45,
    "nearest_surface_below_z": None,          # 下方无接触面
}
Topology = {
    "support(block, _)": False,               # 没有任何支撑
    "supported": False,
}
Semantics = {
    "instruction":  "keep the block on the table",
    "goal":         "support(block, table) == True",
    "status":       "failure",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 只看 Z 高度以为它"在桌上" | Topology | 没有检测 support 的真值条件（必须下方有接触面） |
| 系统认为只要 Z>0.7m 还在桌面高度范围就算支撑 | Topology | 只有高度信息，没有接触检测，会误判 |

---

### 场景 13：手遮住物体

```python
Geometry = {
    "hand_centroid":      (0.5, 0.35, 0.4),
    "object_centroid":    (0.5, 0.35, 0.6),   # 在手后面
    "camera_origin":      (0.5, 0.3, 0),
    "ray_blocked":        True,                # 从相机发出的 ray 被手挡住
}
Topology = {
    "occludes(hand, object)": True,            # ← 遮挡关系，不是距离
}
Semantics = {
    "instruction":  "identify the object",
    "goal":         "occludes(_, object) == False",
    "status":       "perception_blocked",      # perception failure 传导到 semantics
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 手和物体距离近但没有遮挡（侧面）| Topology | 只判距离会误判 occlusion |
| occlusion 导致无法确认 inside/apple 在碗里 | **Semantics** | perception failure 向下传导：geometry 缺了 → topology 判不了 → semantics 无法做 success check |

---

### 场景 14：机器人绕过障碍

```python
Geometry = {
    "start":        (0, 0, 0),
    "goal":         (1, 0, 0),
    "obstacle":     (0.5, 0, 0.4),
    "path":         [(0,0,0), (0.3,0.2,0), (0.5,0.2,0), (0.7,0,0), (1,0,0)],
    "path_length":  1.35,                      # 不是直线距离 1.0
}
Topology = {
    "reachable(start, goal)": True,            # 绕行可到达
    "free_space_components": 1,
}
Semantics = {
    "instruction":  "navigate to the goal",
    "goal":         "reachable(start, goal) == True",
    "status":       "path_found",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 直线距离很小（1.0）判 reachable，实际障碍挡路 | Topology | 不能只看欧氏距离 |
| 绕行路径碰撞了障碍的 corner | Geometry | path 的碰撞检测精度不够 |

---

### 场景 15：线穿过环

```python
Geometry = {
    "line_endpoints":       [(0, 0, 0), (1, 0, 0)],
    "ring_center":          (0.5, 0, 0),
    "ring_radius":          0.05,
    "ring_normal":          (1, 0, 0),
}
Topology = {
    "through(line, ring)": True,              # ← 这不是普通距离关系
    "linking_number":       1,                 # 拓扑不变量：link=1
}
Semantics = {
    "instruction":  "put the thread through the ring",
    "goal":         "through(line, ring) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 线和环距离很近（<0.01），但没穿过 | Topology | near≠through，through 是拓扑关系不是度量关系 |
| linking=1 判正确，但实际 linking=2（绕了两圈）| Topology | linking number 判断失误 |
| 线足够靠近环但没穿过，系统判"任务完成" | Semantics | success check 用了 near 替代 through |

---

### 场景 16：线在环旁边

```python
Geometry = {
    "line_endpoints":     [(0, 0, 0.1), (1, 0, 0.1)],
    "ring_center":        (0.5, 0, 0),
    "distance_line_ring": 0.1,              # 很近
}
Topology = {
    "through(line, ring)": False,           # ← 没穿过
    "linking_number":       0,
}
Semantics = {
    "instruction":  "put the thread through the ring",
    "goal":         "through(line, ring) == True",
    "status":       "failure",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 距离只有 0.1 判 through=True | Topology | 把 metric proximity 当成了 topological linking |

---

### 场景 17：peg 靠近孔

```python
Geometry = {
    "peg_centroid":         (0.5, 0.1, 0.3),
    "hole_centroid":        (0.5, 0.1, 0.2),
    "peg_axis":             (0, 0, -1),       # 朝向孔
    "alignment_angle":      0.05,              # 基本对齐
    "distance_peg_hole":    0.1,
}
Topology = {
    "inside(peg, hole)":    False,            # 还没插进去
    "aligned(peg, hole)":   True,             # 对齐了，但对齐 ≠ inside
}
Semantics = {
    "instruction":  "insert the peg into the hole",
    "goal":         "inside(peg, hole) == True",
    "status":       "preparing",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 对齐了但还没插入，判 inside=True | Topology | 对齐不等于 containment |
| 距离很小但轴线没对齐，也判 inside=False ← 这其实是对的 | Topology | inside 的判断要综合距离+方向+边界 |

---

### 场景 18：peg 插入孔

```python
Geometry = {
    "peg_centroid":       (0.5, 0.1, 0.15),
    "hole_centroid":      (0.5, 0.1, 0.2),
    "peg_axis":           (0, 0, -1),
    "peg_bottom_z":       0.07,
    "hole_bottom_z":      0.03,
}
Topology = {
    "inside(peg, hole)":  True,
    "fit_type":           "tight_fit",
}
Semantics = {
    "instruction":  "insert the peg into the hole",
    "goal":         "inside(peg, hole) == True",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| peg 的 centroid 在 hole 的 centroid 上方但 peg 下端已在 hole 内 | Geometry → Topology | 只看 centroid 会误判，要检查边界/体积 |

---

### 场景 19：抽屉关闭

```python
Geometry = {
    "drawer_joint_angle": 0.0,
    "drawer_linear_displacement": 0.0,
}
Topology = {
    "articulation_state(drawer)": "closed",
    "inside(desk, drawer)": True,
}
Semantics = {
    "instruction":  "close the drawer",
    "goal":         "articulation_state(drawer) == closed",
    "status":       "success",
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| joint angle ≈ 0 但抽屉实际被卡住没完全闭合 | Geometry | 传感器读数对但物理状态不对 |
| articulation state 是连续值，硬判 closed/open 阈值的边缘会不稳定 | Topology | discrete label 需要合理阈值 |

---

### 场景 20：碗口被遮挡

```python
Geometry = {
    "bowl_centroid":  (0.18, 0.14, 0.0),
    "bowl_rim":       [...],
    "obstacle":       (0.2, 0.15, 0.09),     # 挡在碗口前
}
Topology = {
    "occludes(obstacle, bowl_rim)": True,
    "bowl_rim_visible":             False,     # 碗口不可见
}
Semantics = {
    "instruction":  "check if the apple is in the bowl",
    "goal":         "inside(apple, bowl) can be confirmed",
    "status":       "cannot_verify",           # ← perception failure 阻塞语义判断
}
```

| 失败 | 层 | 原因 |
|------|---|------|
| 碗口被遮 → 无法确认 inside(apple, bowl) | **Semantics** | Geometry → Topology → Semantics 的级联失败：缺了几何信息 → 拓扑关系无法判定 → 任务成功条件无法验证 |

---

## 跨场景总结

### 四种核心教训

| 教训 | 对应场景 | 一句话 |
|------|----------|--------|
| **Geometry 变 ≠ Topology 变** | 场景 6、9 | 平移、微开缝：度量变了但结构关系没变 |
| **Geometry 变小 ≠ Topology 变小** | 场景 7、8、12 | 掉落、靠近碗、悬空：一个坐标的小变化可能触发 topology event |
| **near ≠ 拓扑关系** | 场景 8、16、17 | near/above 是 metric relation，inside/through/contact 是 topology relation |
| **perception 失败会级联** | 场景 13、20 | Geometry 缺了 → Topology 判不了 → Semantics 做不了 success check |

### 失败定位速查

```text
坐标/深度/位姿错了          → Geometry failure
关系判断错了（inside/near）  → Topology failure（静态）
没检测到关系变化事件         → Topology failure（动态）
goal 不对 / success check 太松 → Semantics failure
上游缺数据导致下游不可判     → 级联 failure（root cause 在 geometry/topology）
```
