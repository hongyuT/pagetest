# pagetest
# 基于大模型的商品标题和图片审核流程说明

## 1. 功能概述

本项目实现了基于大模型（LLM）的商品标题和图片自动审核机制。系统每天从 `mara_ai_pict` 数据库表中读取待审核的商品信息，调用大模型接口对商品图片和标题进行内容审核，并将审核结果保存到 `mara_ai_pict_result` 数据库表中。对于审核不通过的商品，会自动插入到 `mara_rec_pict` 表供人工复核。

---

## 2. 数据表结构

### 2.1 输入表：mara_ai_pict

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint | 主键，自增 |
| nid | bigint | 商品ID |
| pict_url | text | 图片链接 |
| title | varchar(200) | 商品标题 |
| cnt | int | 计数 |
| ds | varchar(20) | 业务日期 |
| source | varchar(20) | 来源表 |
| biz | varchar(20) | 业务来源（默认tt） |
| status | int | 审核状态：0-未审核，2-处理成功，3-处理失败 |

### 2.2 输出表：mara_ai_pict_result

| 字段名 | 类型 | 说明 |
|--------|------|------|
| id | bigint(20) | 主键，自增 |
| gmt_create | datetime | 创建时间 |
| gmt_modified | datetime | 修改时间 |
| llm_reason | varchar(256) | 大模型审核理由 |
| llm_tag | int(11) | 大模型打标 |
| llm_pict | varchar(256) | 商品图片 |
| status | int(11) | 是否合规：0-合规，1-不合规 |
| title | varchar(256) | 商品标题 |
| nid | varchar(128) | 商品ID |
| ds | varchar(128) | 日期 |

### 2.3 人工复核表：mara_rec_pict

| 字段名 | 类型 | 说明 |
|--------|------|------|
| nid | bigint | 商品ID |
| pict_url | varchar | 图片链接 |
| title | varchar | 商品标题 |
| ds | varchar | 业务日期 |
| source | varchar | 来源 |
| biz | varchar | 业务来源 |

---

## 3. 核心组件

### 3.1 组件列表

| 组件类型 | 类名 | 文件路径 | 职责 |
|----------|------|----------|------|
| 调度任务 | LlmReviewProcessor | mara-service/src/main/java/com/taobao/mara/scheduler/LlmReviewProcessor.java | SchedulerX定时任务入口 |
| 服务接口 | LlmReviewService | mara-service/src/main/java/com/taobao/mara/service/LlmReviewService.java | 审核服务接口定义 |
| 服务实现 | LlmReviewServiceImpl | mara-service/src/main/java/com/taobao/mara/service/impl/LlmReviewServiceImpl.java | 审核流程核心实现 |
| 外部服务接口 | IdeaLabService | mara-service/src/main/java/com/taobao/mara/service/IdeaLabService.java | LLM接口调用接口 |
| 外部服务实现 | IdeaLabServiceImpl | mara-service/src/main/java/com/taobao/mara/service/impl/IdeaLabServiceImpl.java | LLM接口调用实现 |
| 配置管理 | LlmDiamond | mara-service/src/main/java/com/taobao/mara/config/LlmDiamond.java | Diamond配置管理 |
| 数据对象 | MaraAiPictDO | mara-service/src/main/java/com/taobao/mara/domain/MaraAiPictDO.java | 输入表数据对象 |
| 数据传输对象 | LlmReviewDataDTO | mara-service/src/main/java/com/taobao/mara/dto/LlmReviewDataDTO.java | LLM请求参数 |
| 数据传输对象 | LlmResultDataDTO | mara-service/src/main/java/com/taobao/mara/dto/LlmResultDataDTO.java | LLM响应结果 |
| 数据传输对象 | LlmConfigDTO | mara-service/src/main/java/com/taobao/mara/dto/LlmConfigDTO.java | LLM配置信息 |
| 数据访问 | MaraAiPictMapper | mara-service/src/main/java/com/taobao/mara/mapper/mara/MaraAiPictMapper.java | 输入表数据访问 |
| 数据访问 | MaraAiPictResultMapper | mara-service/src/main/java/com/taobao/mara/mapper/mara/MaraAiPictResultMapper.java | 输出表数据访问 |

---

## 4. 审核流程详解

### 4.1 整体流程图

```
┌─────────────────┐
│  SchedulerX     │
│  定时触发       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ LlmReviewProcessor │
│ (调度任务入口)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  processReview  │
│ (主处理流程)    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│读取待审│ │线程池  │
│核数据  │ │并发处理│
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌─────────────────┐
│   processItem   │
│  (单条处理)     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│调用LLM │ │重试机制│
│接口    │ │(最多3次)│
└───┬────┘ └────────┘
    │
    ▼
┌─────────────────┐
│  parseLlmResponse│
│  (解析响应)     │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│保存审核│ │不合规  │
│结果    │ │插入人工│
└────────┘ │复核表  │
           └────────┘
```

### 4.2 详细流程说明

#### 步骤1：定时任务触发

| 项目 | 说明 |
|------|------|
| 触发器 | SchedulerX定时任务框架 |
| 任务类 | LlmReviewProcessor |
| 调度周期 | 每天执行（可通过SchedulerX配置） |
| 任务参数 | 支持传入ds参数指定业务日期，如 `ds=20240305` |

#### 步骤2：数据读取

| 项目 | 说明 |
|------|------|
| 数据来源 | mara_ai_pict表 |
| 查询条件 | ds = 指定日期 AND status = 0（未审核） |
| 批次大小 | 每次读取1000条（BATCH_SIZE = 1000） |
| 循环读取 | 循环查询直到没有未审核数据 |

#### 步骤3：并发处理

| 项目 | 说明 |
|------|------|
| 并发控制 | 使用线程池（ExecutorService） |
| 最大并发数 | 默认3个线程，可通过Diamond配置动态调整 |
| 同步机制 | 使用BlockingQueue等待每批任务完成 |
| 状态管理 | 使用AtomicInteger统计提交和完成的任务数 |

#### 步骤4：单条数据处理

| 子步骤 | 说明 |
|--------|------|
| 数据转换 | 将MaraAiPictDO转换为LlmReviewDataDTO |
| LLM调用 | 调用IdeaLabService.postRequestRunModel() |
| 重试机制 | 失败时最多重试3次，每次间隔递增（指数退避） |
| 响应解析 | 解析JSON响应，提取审核结果 |
| 结果保存 | 保存到mara_ai_pict_result表 |
| 状态更新 | 更新mara_ai_pict表状态为成功(2)或失败(3) |
| 人工复核 | 如审核不通过(status≠0)，插入mara_rec_pict表 |

---

## 5. 数据流说明

### 5.1 数据流转图

```
┌─────────────────────────────────────────────────────────────────┐
│                         数据流转图                               │
└─────────────────────────────────────────────────────────────────┘

  ┌──────────────┐
  │ mara_ai_pict │
  │  (输入表)    │
  │  status=0    │
  └──────┬───────┘
         │ 1. 查询未审核数据
         │    SELECT * WHERE ds=? AND status=0 LIMIT 1000
         ▼
  ┌──────────────┐
  │ LlmReview    │
  │ ServiceImpl  │
  │ (批量处理)   │
  └──────┬───────┘
         │ 2. 提交到线程池
         │    executorService.submit()
         ▼
  ┌──────────────┐
  │  ThreadPool  │
  │  (并发处理)  │
  └──────┬───────┘
         │ 3. 单条处理
         │    processItem()
         ▼
  ┌──────────────┐
  │ IdeaLab      │
  │ ServiceImpl  │
  │ (调用LLM)    │
  └──────┬───────┘
         │ 4. HTTP POST请求
         │    调用大模型API
         ▼
  ┌──────────────┐
  │   LLM API    │
  │ 大模型服务   │
  └──────┬───────┘
         │ 5. 返回审核结果(JSON)
         ▼
  ┌──────────────┐
  │  parseLlm    │
  │  Response    │
  │ (解析响应)   │
  └──────┬───────┘
         │ 6. 解析结果
         │    status/reason/tag
         ▼
       ┌─┴─┐
       │   │
       ▼   ▼
  ┌────────┐  ┌────────┐
  │status=0│  │status≠0│
  │ 合规   │  │不合规  │
  └───┬────┘  └────┬───┘
      │            │
      ▼            ▼
┌──────────┐  ┌──────────┐
│mara_ai_  │  │mara_rec_ │
│pict_result│  │pict      │
│(审核结果)│  │(人工复核)│
└──────────┘  └──────────┘
      │            │
      └──────┬─────┘
             ▼
      ┌──────────┐
      │mara_ai_  │
      │pict      │
      │更新status│
      │2=成功    │
      │3=失败    │
      └──────────┘
```

### 5.2 数据流详细说明

| 步骤 | 数据源 | 目标 | 操作 | 说明 |
|------|--------|------|------|------|
| 1 | mara_ai_pict | LlmReviewService | 查询 | 按ds和status=0查询，每次1000条 |
| 2 | LlmReviewService | ThreadPool | 提交任务 | 为每条数据创建处理任务 |
| 3 | ThreadPool | processItem | 执行 | 单条数据处理 |
| 4 | processItem | IdeaLabService | 调用 | 构造请求参数调用LLM |
| 5 | IdeaLabService | LLM API | HTTP POST | 发送商品信息和大模型交互 |
| 6 | LLM API | IdeaLabService | 响应 | 返回JSON格式审核结果 |
| 7 | IdeaLabService | parseLlmResponse | 解析 | 提取status、reason、tag等字段 |
| 8 | parseLlmResponse | mara_ai_pict_result | 插入 | 保存审核结果 |
| 9 | parseLlmResponse | mara_rec_pict | 插入 | 如不合规(status≠0)，插入人工复核表 |
| 10 | processItem | mara_ai_pict | 更新 | 更新status为2(成功)或3(失败) |

---

## 6. LLM接口调用详情

### 6.1 请求参数

| 参数名 | 类型 | 说明 |
|--------|------|------|
| empId | String | 员工ID（固定值158763） |
| sessionId | String | 会话ID（UUID生成） |
| stream | Boolean | 是否流式输出（false） |
| returnRunLog | Boolean | 是否返回运行日志（false） |
| variableMap | Object | 变量映射 |
| variableMap.nid | String | 商品ID |
| variableMap.title | String | 商品标题 |
| variableMap.pict_url | String | 图片URL |
| mediaEntities | Array | 多模态数据 |
| mediaEntities[0].content | String | 图片URL |

### 6.2 请求示例

```json
{
  "empId": "158763",
  "sessionId": "uuid-generated-string",
  "stream": false,
  "returnRunLog": false,
  "variableMap": {
    "nid": "123456789",
    "title": "商品标题示例",
    "pict_url": "https://img.alicdn.com/imgextra/xxx.jpg"
  },
  "mediaEntities": [
    {
      "content": "https://img.alicdn.com/imgextra/xxx.jpg"
    }
  ]
}
```

### 6.3 响应解析

| 字段 | 说明 |
|------|------|
| success | 接口调用是否成功 |
| data.content | LLM返回的JSON数组字符串 |
| nid | 商品ID（用于校验） |
| pict_url | 图片URL |
| title | 商品标题 |
| reason | 审核理由 |
| status | 审核状态：0-合规，1-不合规 |
| tag | 标签分类 |

### 6.4 图片URL处理

| 处理逻辑 | 说明 |
|----------|------|
| 原始URL | 可能为相对路径或绝对路径 |
| 转换规则 | 如不以`https://img.alicdn.com`开头，则添加前缀`https://img.alicdn.com/imgextra/` |
| 目的 | 确保图片URL可正常访问 |

---

## 7. 配置管理

### 7.1 Diamond配置

| 配置项 | Data ID | Group | 说明 |
|--------|---------|-------|------|
| LLM配置 | com.mara.llm | hjb | 大模型API配置 |

### 7.2 配置内容

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| apiUrl | String | "" | LLM API地址 |
| apiKey | String | "" | API认证密钥（X-AK头） |
| maxConcurrency | int | 3 | 最大并发数 |

### 7.3 动态配置更新

| 特性 | 说明 |
|------|------|
| 实时监听 | 使用Diamond监听器实时接收配置变更 |
| 线程池调整 | 配置变更时动态调整线程池大小 |
| 原子引用 | 使用AtomicReference保证线程安全 |

---

## 8. 异常处理与重试机制

### 8.1 重试策略

| 项目 | 说明 |
|------|------|
| 最大重试次数 | 3次（MAX_RETRY_COUNT = 3） |
| 重试间隔 | 指数退避：1秒、2秒、3秒 |
| 重试场景 | LLM接口调用失败时 |
| 中断处理 | 线程中断时抛出RuntimeException |

### 8.2 异常处理流程

| 异常类型 | 处理方式 |
|----------|----------|
| LLM调用失败 | 重试3次后仍失败，标记为处理失败(status=3) |
| 响应解析失败 | 抛出RuntimeException，标记为处理失败 |
| 商品ID不匹配 | 记录错误日志，抛出异常 |
| 数据库更新失败 | 记录错误日志，不影响其他数据 |
| 插入人工复核表失败 | 记录错误日志，不影响主流程 |

---

## 9. 关键代码说明

### 9.1 核心处理流程

```java
// LlmReviewServiceImpl.processReview()
public void processReview(JobContext jobContext) {
    // 1. 初始化线程池
    LlmConfigDTO config = LlmDiamond.getUsedLlmConfig();
    executorService = Executors.newFixedThreadPool(config.getMaxConcurrency());
    
    // 2. 循环读取数据
    while (true) {
        List<MaraAiPictDO> unreviewedItems = maraAiPictMapper.selectUnreviewedItems(ds, BATCH_SIZE);
        if (unreviewedItems.isEmpty()) break;
        
        // 3. 提交到线程池并发处理
        for (MaraAiPictDO item : unreviewedItems) {
            executorService.submit(() -> {
                try {
                    processItem(item);
                } finally {
                    completionQueue.offer(true);
                }
            });
        }
        
        // 4. 等待本批任务完成
        for (int i = 0; i < unreviewedItems.size(); i++) {
            completionQueue.take();
        }
    }
}
```

### 9.2 单条数据处理

```java
// LlmReviewServiceImpl.processItem()
private void processItem(MaraAiPictDO item) {
    // 1. 数据转换
    LlmReviewDataDTO reviewData = new LlmReviewDataDTO();
    reviewData.setNid(String.valueOf(item.getNid()));
    reviewData.setTitle(item.getTitle());
    reviewData.setPictUrl(item.getPictUrl());
    
    // 2. 调用LLM（带重试）
    String response = callLlmWithRetry(reviewData);
    
    // 3. 解析响应
    LlmResultDataDTO resultData = parseLlmResponse(response, item);
    
    // 4. 保存结果
    maraAiPictResultMapper.insert(resultData);
    
    // 5. 更新状态
    maraAiPictMapper.updateItemStatus(item.getId(), STATUS_SUCCESS);
    
    // 6. 不合规时插入人工复核表
    if (!"0".equals(resultData.getStatus())) {
        insertIntoMaraRecPict(item, resultData);
    }
}
```

### 9.3 LLM调用（带重试）

```java
// LlmReviewServiceImpl.callLlmWithRetry()
private String callLlmWithRetry(LlmReviewDataDTO reviewData) {
    int retryCount = 0;
    while (retryCount < MAX_RETRY_COUNT) {
        try {
            return ideaLabService.postRequestRunModel(reviewData);
        } catch (Exception e) {
            retryCount++;
            if (retryCount >= MAX_RETRY_COUNT) {
                throw new RuntimeException("调用LLM接口失败", e);
            }
            // 指数退避
            Thread.sleep(1000 * retryCount);
        }
    }
}
```

---

## 10. 总结

### 10.1 核心特性

| 特性 | 实现方式 |
|------|----------|
| 定时触发 | SchedulerX定时任务 |
| 批量处理 | 每次读取1000条，循环处理 |
| 并发控制 | 线程池，默认3并发，可动态配置 |
| 失败重试 | 最多3次，指数退避 |
| 配置管理 | Diamond动态配置 |
| 人工复核 | 自动插入不合规数据到复核表 |

### 10.2 数据完整性保障

| 机制 | 说明 |
|------|------|
| 状态管理 | 每条数据都有明确的状态流转 |
| 异常隔离 | 单条数据处理失败不影响其他数据 |
| 幂等性 | 通过status字段避免重复处理 |
| 数据校验 | 校验返回的商品ID与请求是否匹配 |

### 10.3 可扩展性

| 扩展点 | 说明 |
|--------|------|
| 并发数调整 | 通过Diamond配置实时调整 |
| 批次大小 | 修改BATCH_SIZE常量 |
| 重试策略 | 修改MAX_RETRY_COUNT和退避时间 |
| LLM接口 | 通过配置切换不同的API地址和密钥 |
