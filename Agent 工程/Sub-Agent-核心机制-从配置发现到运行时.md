# Claude Code / Harness 中 Sub Agent 的核心机制

下面我会去阐述 Harness 中 Sub Agent 的机制。

如果简单来说，我目前对 Sub Agent 的理解，可以把它拆成三大部分：

> **注册与发现 → 创建与配置 → Runtime 运行**

值得注意的是，Sub Agent 并不是真的像启动一个独立产品一样，提前创建好一个 Agent 放在那里等待使用。

很多时候，它的本质仍然是一次 **Tool Call**。

也就是说，Parent Agent 在自己的 Agent Loop 中发现：

> 当前有一部分工作比较独立，适合从主任务中剥离出来，交给另外一个 Agent 单独完成。

于是 Parent Agent 调用类似：

```
delegate_task
subagent
spawn_agent
```

这样的工具。

这个工具背后，再由 Harness Runtime 创建一个 Child Agent，让 Child 自己运行一个独立的 Agent Loop，最后把结果返回给 Parent Agent。

所以整个过程大概就是：

```
Parent Agent
    ↓
发现任务适合委派
    ↓
寻找 / 定义一个 Sub Agent
    ↓
配置 Context / Prompt / Tools / MCP / Model / Permission
    ↓
创建 Child Agent
    ↓
Child 自己运行 Agent Loop
    ↓
得到 Result
    ↓
返回 Parent Agent
```

这里我先不去考虑 Child Agent 具体是怎么运行的，先来看前面的 **配置、发现和创建**。

## 一、Sub Agent 的配置与发现

现在比较主流的一种方式，是提前通过 Markdown 文件定义 Agent。

例如：

```
researcher.md
coder.md
reviewer.md
```

另外也可以通过临时 JSON 定义。

但是这里有一个非常重要的认识：

> **Markdown 文件本身不是 Agent，它描述的是 Agent Profile。**

例如一个 `researcher.md`，里面实际上是在定义：

```
这个 Agent 是干什么的
System Prompt 是什么
允许使用哪些 Tools
使用什么 Model
有什么能力边界
```

真正运行时，Parent Agent 找到这个 Profile，然后 Runtime 根据这个 Profile 创建真正的 Child Agent Instance。

所以：

```
researcher.md
      ↓
Agent Profile
      ↓
Agent Registry
      ↓
Parent 调用 SubAgent Tool
      ↓
创建 Child Instance
```

Claude Code / Pi 这一类设计比较接近这种方式。

Hermes 的思路稍微不同一点。

Hermes 可以更加动态地创建 Child Agent，也就是直接动态 `new` 一个 Child `AIAgent`。

因此这里就出现了一个非常重要的问题：

> 如果 Agent 可以动态创建新的 Agent，那么是不是意味着 LLM 可以自己随便决定这个 Agent 有什么权限、Tool、MCP 和运行环境？

答案不是。

这也是 Dynamic Sub-agent 里面非常重要的一个架构思想：

> **Dynamic Sub-agent 动态的是“任务实例”，不是“安全边界”。**

Parent LLM 可以决定：

```
我要创建一个 Child
让它去研究 auth 模块
```

但是 Parent LLM 并不能随意决定：

```
给它 database_write
给它 root shell
让它无限递归创建 Sub Agent
让它访问所有 MCP
```

这些东西最终还是 Runtime 决定。

所以动态创建 Sub Agent 的完整设计，更准确的是：

```
Parent LLM
    ↓
SubAgentRequest
    ↓
SubAgent Policy
    ↓
SubAgent Factory
    ↓
Child Agent Instance
```

其中 Parent LLM 负责提出：

```
goal
context
role/profile
```

而 Runtime 负责控制真正的能力边界。

这里还有一个原则：

> **Parent Agent 的权限通常是 Child Agent 权限的上界。**

也就是说 Parent 自己都没有的能力，不能因为它创建了一个 Child，就让 Child 获得这个能力。

SubAgent Policy 的规则通常来自三个地方：

```
系统级 Policy
Harness 开发者定义的全局限制

Agent Profile Policy
researcher / coder / reviewer 自身的能力边界

Runtime Context Policy
当前 user / tenant / parent agent 的权限
```

最终 Runtime 根据这些规则，计算出来一个真正可以运行的 Child Agent 配置，然后交给 Factory 创建。

------

# 二、Child Agent 的资源是怎么分配的

Child Agent 创建出来之后，就涉及：

```
Context
Tools
MCP
Permission
Model
Workspace
```

这些东西怎么给它。

这里同样有一个重要原则：

> **通常不是直接复制 Parent 的全部资源，而是经过 Runtime 继承、裁剪、重新装配。**

### Context

Sub Agent 一般会创建一个新的 Context。

通常只包含：

```
System Prompt
+
当前任务 goal
+
Parent 显式传递的必要 context
+
必要的文件 / workspace 信息
```

例如 Parent 已经进行了 30 轮对话。

Child 并不一定需要知道这 30 轮。

Parent 只需要告诉它：

```
去分析 auth 模块，
重点检查 token refresh，
相关代码在 xxx。
```

Child 就可以开始工作。

所以：

```
Parent Full Context
       ↓
选择必要信息
       ↓
Child Context
```

而不是：

```
Parent Full Context
       ↓
全部复制
       ↓
Child
```

这也是 Sub Agent 和 Fork Agent 一个非常重要的区别。

## 三、Sub Agent 和 Fork Agent

Sub Agent 的 Context 通常是：

> **新建 Context，只拿 Parent 显式传入的任务信息。**

而 Fork Agent 是：

> **从 Parent 当前状态做一个 Context Snapshot，然后从这个节点创建新的执行分支。**

所以：

```
Sub Agent
= Task Delegation

Fork Agent
= Context / State Branching
```

例如：

```
Parent 已经分析问题 30 轮
```

现在想：

```
让一个 Agent 单独去研究数据库问题
```

这个更适合 Sub Agent。

因为只需要把数据库相关任务交给它。

但是如果：

```
当前已经形成大量中间结论，
现在想同时走方案 A 和方案 B
```

这种情况下更适合 Fork。

因为两个新的 Agent 都需要继承前面已经形成的完整 Context。

------

# 四、Tools 与 MCP 的分配

Tools 也是类似的思想：

> **继承上限 + Policy 裁剪。**

例如 Parent 拥有：

```
read
write
bash
sql
send_email
```

但是现在创建的是一个 Researcher Agent。

经过：

```
Parent Capability
∩
Researcher Profile
∩
Permission
∩
SubAgent Policy
```

最后 Child 可能只有：

```
read
grep
bash
```

所以 Child 会重新形成自己的：

```
Tool Snapshot
```

而不是直接复制 Parent 当前所有 Tool。

MCP 也是一样。

例如 Parent 当前连接了：

```
Filesystem MCP
GitHub MCP
Database MCP
Slack MCP
```

Researcher Child 可能只获得：

```
Filesystem MCP
GitHub MCP
```

Database Agent 才会获得：

```
Database MCP
```

所以可以把这一部分概括成：

> **Context 是按任务传递，Tools 是按权限裁剪，MCP 是按能力范围挂载。**

最终 Runtime 会形成一个类似：

```
ResolvedAgentSpec
```

这里面包含：

```
Prompt
Context
Tools
MCP
Model
Permission
Runtime Limit
```

然后 Factory 根据这个 Spec 创建 Child Agent。

------

# 五、下面进入真正的 Runtime

到这里，其实 Sub Agent 的：

```
发现
配置
权限
Context
Tools
MCP
创建
```

这些东西我已经大概理解清楚了。

下面真正进入 Runtime。

也就是：

> Child Agent 创建以后，到底是在哪里跑的？

这个问题需要先区分两个完全不同的东西：

```
Child Agent Loop 在哪里运行？

Tool 真正在哪里执行？
```

这两件事情不是一回事。

Child Agent 本身，本质上是另外启动一个：

```
Agent Loop
```

它独立进行：

```
LLM
 ↓
Tool Call
 ↓
Result
 ↓
LLM
 ↓
...
 ↓
Final Result
```

目前我了解到 Child Loop 大概存在三种运行方式：

```
同进程中的独立 Agent Loop

独立子进程

Remote Worker / Container
```

也就是说：

```
Parent Agent
      ↓
SubAgent Runtime
      ↓
Child Agent Loop
      ↓
独立运行
      ↓
Final Result
      ↓
Parent Agent
```



