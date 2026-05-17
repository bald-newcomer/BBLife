# 节点
## 基本节点
```dot
digraph Example {
    // 简单节点
    A;

    // 带标签的节点
    B [label="节点B"];  

    // 包含空格的节点名
    "Node C";   
}
```

## 节点基本属性
```dot
digraph NodeAttributes {
    A [shape=box, color=red, style=filled, fillcolor=lightblue];
    B [shape=circle, width=1.5, height=1.0];
    C [shape=ellipse, label="重要节点", fontsize=16];
    D [shape=diamond, sides=4, skew=0.5];
}
```
## 常用节点形状：
* circle, box, ellipse, diamond
* triangle, plaintext, record, Mrecord
* doublecircle, doubleoctagon

# 边
## 基本边
```dot
digraph EdgeExamples {
    // 单向边
    A -> B;
    
    // 双向边
    C -> D [dir=both];
    
    // 无向边（不推荐在 digraph 中使用）
    E -> F [dir=none];
}
```

## 边属性
```dot
digraph EdgeAttributes {
    rankdir=LR;  // 从左到右 (Left to Right)
    A -> B [label="步骤1", color=red, style=dashed];
    B -> C [label="步骤2", color=blue, style=bold];
    C -> D [arrowhead=vee, arrowsize=2.0];
    D -> E [label="重要", fontcolor=green,fontsize=12];
}
```
# 图形布局属性
## 方向控制
```dot
digraph Layout {
    rankdir=TB;  // 从上到下 (Top to Bottom)
    // rankdir=LR;  // 从左到右 (Left to Right)
    // rankdir=BT;  // 从下到上
    // rankdir=RL;  // 从右到左
    
    A -> B -> C;
    D -> E -> F;
}
```
## 子图
```dot
digraph SubgraphExample {
    rankdir=LR;
    
    subgraph cluster_0 {
        label="流程组A";
        color=blue;
        A -> B -> C;
    }
    
    subgraph cluster_1 {
        label="流程组B";
        color=red;
        D -> E -> F;
    }
    
    C -> D;  // 连接两个子图
}
```
# 复杂示例

## 软件架构图
```dot
digraph SoftwareArchitecture {
    rankdir=TB;
    node [shape=rect, style=rounded];
    
    // 用户层
    User [shape=circle, style=filled, fillcolor=lightblue];
    
    // 前端层
    subgraph cluster_frontend {
        label="前端服务";
        color=lightblue;
        "Web Server" -> "Load Balancer";
        "Load Balancer" -> "App Server 1";
        "Load Balancer" -> "App Server 2";
    }
    
    // 后端层
    subgraph cluster_backend {
        label="后端服务";
        color=lightgreen;
        "API Gateway";
        "Auth Service";
        "User Service";
        "Product Service";
    }
    
    // 数据层
    subgraph cluster_data {
        label="数据层";
        color=orange;
        "Redis Cache" [shape=cylinder];
        "MySQL DB" [shape=cylinder];
        "MongoDB" [shape=cylinder];
    }
    
    // 连接关系
    User -> "Web Server";
    "App Server 1" -> "API Gateway";
    "App Server 2" -> "API Gateway";
    "API Gateway" -> {"Auth Service", "User Service", "Product Service"};
    "Auth Service" -> "Redis Cache";
    "User Service" -> "MySQL DB";
    "Product Service" -> "MongoDB";
}
```

## 状态机图
```dot
digraph StateMachine {
    rankdir=LR;
    node [shape=circle, width=1.2];
    
    Start [shape=point, width=0.1];
    Idle [style=filled, fillcolor=lightgreen];
    Processing [style=filled, fillcolor=yellow];
    Error [style=filled, fillcolor=red];
    Success [shape=doublecircle, style=filled, fillcolor=lightblue];
    
    Start -> Idle;
    Idle -> Processing [label="request"];
    Processing -> Success [label="success"];
    Processing -> Error [label="error"];
    Error -> Idle [label="retry"];
    Success -> Idle [label="reset"];
}
```


