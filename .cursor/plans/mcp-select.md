下面先给出结论，再对关键环节做简要说明。

# 结论
AI（大模型）**并不决定**应该使用哪个 MCP Server，**由 Client（即用户或接入层）在启动时明确指定**要和哪些 MCP Server 建立连接。AI 的职责只是在这些已连接的服务器所提供的工具列表中，基于需求去**挑选并调用**合适的工具（tool）。

# 详细流程

1. 客户端（Client）配置 MCP Server  
   - 在 Client 的配置文件或启动参数里，用户指定要连接的一组 MCP Server（如高德地图、天气、翻译等）。  
   - Client 与这些服务器建立长连接／SSE 连接，准备好后续的协议交互。

2. Client 向每个 MCP Server 拉取“可用工具列表”  
   - 调用 `session.list_tools()`，服务器返回名为 `name`、带描述 `description`、输入 Schema `inputSchema` 的一系列工具。  
   - Client 将所有工具汇总，并在后续构造 Prompt 时一并提供给大模型。

3. Client 构造 Prompt 并发送给大模型  
   - Prompt 中包含用户原始问题 + “以下是可用工具及其描述” 的清单。  
   - 示例：
     > “我想做五一南京三日游规划，注意天气和公共交通。  
     > 工具列表：  
     > 1. maps_weather：查询城市天气……  
     > 2. maps_direction_driving：驾车路线规划……”  

4. 大模型决定是否以及如何使用工具  
   - 分析用户需求，若仅需文本回答，则直接输出；  
   - 若需要外部数据或操作，则匹配工具描述，生成 `tool_use` 指令，指定工具名和调用参数。

5. Client 收到 `tool_use` 指令后，通过 MCP 协议调用对应工具  
   - 执行 `session.call_tool(tool_name, tool_args)`，请求 MCP Server 运行业务逻辑。  
   - MCP Server 将执行结果（如天气数据、路线规划结果）返回给 Client。

6. 大模型整合工具返回结果，生成最终回答  
   - Client 把工具输出送回模型，模型在此基础上补充说明，形成对用户的最终回复。

# MCP 在其中的角色
- **协议规范者**：定义了 Client ↔ Server 的发现（list_tools）、调用（call_tool）和结果返回格式。  
- **中立传输者**：不参与决策，只保证工具能力被正确描述并被执行。

---

通过上述流程可以清晰地看到，**MCP Server 的选定**由 Client 在启动配置阶段完成，AI 只接收来自这些服务器的工具列表，并在运行时决定调用哪些具体工具。