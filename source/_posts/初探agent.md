---
title: 初探 Agent
date: 2025-08-01 17:31:29
categories: "LLM"
comments: true
featured_image: pic1.jpg
tags:
- LLM
- Agent
---

<!-- no node  -->

<!-- more -->

这个月基本围绕着之前的 WebContainer 去实现一个 Agent ，也算是负责 Agent 核心开发了。

Agent 的设计从最初简单的工具调用、冗长上下文演变成任务能够分模块执行，自由的编排各个任务的执行顺序（代码层面，非UI拖拽）。

从原先一大坨业务固定的代码演变成各个任务只需关注自身和其上下文即可，这个月可算是从基本实现到升级解耦全做了个遍。

目前设计的 Agent 架构信息如下：

## Agent 架构设计图

![architecture](./mermaid-diagram.png)

```mermaid
graph TB
    subgraph "用户交互层"
        User[👤 用户]
        WebUI[🌐 Web界面]
    end
    
    subgraph "核心引擎层"
        MainEngine[🔄 主循环引擎]
        TaskScheduler[📋 任务调度器]
        TaskExecutor[⚡ 任务执行器]
        WaitHandler[⏳ 等待处理器]
        ExceptionHandler[🚨 异常处理器]
        Logger[📝 日志模块]
    end
    
    subgraph "会话处理层"
        SessionProcessor[💬 会话处理器]
        SessionRouter[🔀 会话路由]
        SSEOutput[📡 SSE流输出]
    end
    
    subgraph "工具管理层"
        ToolManager[🛠️ 工具管理器]
        ToolExecutor[⚙️ 工具执行]
        ParamValidator[✅ 参数验证]
        ExecutionOptimizer[🚀 执行优化]
        
        subgraph "具体工具"
            FileRWTool[📁 文件读写工具]
            FileSearchTool[🔍 文件搜索工具]
            FileDiffTool[📝 文件搜索替换/DIFF工具]
        end
    end
    
    subgraph "知识管理层"
        RAG[🧠 RAG系统]
        KnowledgeProtocol[📚 知识维护协议规范]
    end
    
    subgraph "模型服务层"
        ModelManager[🤖 模型管理器]
        LLMService[🎯 大语言模型服务]
    end
    
    %% 用户交互流
    User --> WebUI
    WebUI --> SessionProcessor
    
    %% 会话处理流
    SessionProcessor --> SessionRouter
    SessionProcessor --> SSEOutput
    SessionRouter --> MainEngine
    
    %% 主引擎内部流
    MainEngine --> TaskScheduler
    TaskScheduler --> TaskExecutor
    TaskExecutor --> WaitHandler
    TaskExecutor --> ExceptionHandler
    MainEngine --> Logger
    
    %% 工具管理流
    TaskExecutor --> ToolManager
    ToolManager --> ToolExecutor
    ToolManager --> ParamValidator
    ToolManager --> ExecutionOptimizer
    
    ToolExecutor --> FileRWTool
    ToolExecutor --> FileSearchTool
    ToolExecutor --> FileDiffTool
    
    %% 知识和模型服务
    RAG --> KnowledgeProtocol
    TaskExecutor --> ModelManager
    TaskExecutor --> RAG
    ModelManager --> LLMService
    
    %% 反馈流
    SSEOutput --> WebUI
    WaitHandler --> SessionProcessor
    ExceptionHandler --> Logger
    Logger -.-> User
    
    %% 样式定义
    classDef userLayer fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef coreLayer fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    classDef sessionLayer fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef toolLayer fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef knowledgeLayer fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    classDef modelLayer fill:#f1f8e9,stroke:#689f38,stroke-width:2px
    classDef toolItem fill:#fff8e1,stroke:#ff8f00,stroke-width:1px
    
    class User,WebUI userLayer
    class MainEngine,TaskScheduler,TaskExecutor,WaitHandler,ExceptionHandler,Logger coreLayer
    class SessionProcessor,SessionRouter,SSEOutput sessionLayer
    class ToolManager,ToolExecutor,ParamValidator,ExecutionOptimizer toolLayer
    class RAG,KnowledgeProtocol knowledgeLayer
    class ModelManager,LLMService modelLayer
    class FileRWTool,FileSearchTool,FileDiffTool toolItem
```

## 架构层次说明

### 🔄 核心引擎层
- **主循环引擎**: 整个Agent的核心控制器，协调各个模块的工作
- **任务调度器**: 负责任务的排队、优先级管理和调度执行
- **任务执行器**: 具体执行各种任务，包括工具调用和模型推理
- **等待处理器**: 处理需要用户交互的场景，如确认、回复等
- **异常处理器**: 统一的异常捕获和处理机制
- **日志模块**: 记录系统运行状态和调试信息

### 💬 会话处理层
- **会话处理器**: 管理用户会话的生命周期
- **会话路由**: 根据请求类型路由到不同的处理逻辑
- **SSE流输出**: 实现实时的流式响应输出

### 🛠️ 工具管理层
- **工具管理器**: 统一管理所有可用工具
- **工具执行**: 安全地执行各种工具操作
- **参数验证**: 确保工具调用参数的正确性
- **执行优化**: 优化工具执行性能和资源使用

### 🧠 知识管理层
- **RAG系统**: 检索增强生成，提供上下文相关的知识
- **知识维护协议**: 规范化的知识更新和维护机制

### 🤖 模型服务层
- **模型管理器**: 管理不同的AI模型实例
- **大语言模型服务**: 提供统一的模型推理接口

## 数据流向

1. **用户请求**: 用户通过Web界面发起请求
2. **会话处理**: 会话处理器接收并路由请求
3. **任务调度**: 主循环引擎调度相应的任务
4. **工具执行**: 根据需要调用各种工具完成任务
5. **模型推理**: 利用大语言模型进行智能处理
6. **结果返回**: 通过SSE流实时返回处理结果

这种架构设计实现了模块化、可扩展的Agent系统，各个组件职责清晰，便于维护和升级。

可以说整个月是游走在 Agent 开发和 Prompt 调试之间，以开发、产品多重角色优化项目中。
