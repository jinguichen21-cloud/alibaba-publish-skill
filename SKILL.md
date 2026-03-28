---
name: alibaba-publish-skill
description: 阿里国际站发品工具。通过浏览器会话验证登录状态，触发搬品任务（当前仅支持URL搬品），并轮询获取最终发品结果。
---

# 阿里国际站发品 SKILL

## 1. 工具概述

本工具专为阿里国际站商品团队设计，用于将原有 URL 发品能力 Agent 化。
**核心原则**：**直接调用后端 API 接口执行任务**，不要直接操作页面！仅在 API 不可用时操作页面！
**核心流程**：
1.  **登录验证**：自动检测浏览器会话状态，未登录时引导用户完成认证。
2.  **参数校验**：校验 URL 合法性及账号资源配额（搬品前预检）。
3.  **任务触发**：调用批量发品接口，提交待发布的商品 URL 列表及相关配置。
4.  **结果轮询**：异步监控任务进度，直至任务成功、失败或超时。
5.  **结果呈现**：根据轮询与明细查询结果，生成**任务概况表**与**任务明细表**。

## 2. 环境与前置要求

*   **浏览器环境**：必须使用 `browser` 工具操作阿里国际站域名。**严禁新建浏览器账号**，必须复用用户当前已登录的常用浏览器会话。
*   **目标域名**：
    *   登录页：`https://login.alibaba.com`
    *   发品服务：`https://post.alibaba.com`
*   **登录状态**：用户必须处于已登录状态。若未登录，需拉起 `https://login.alibaba.com/newlogin/icbuLogin.htm` 让用户完成登录，并通过 `Exec` 工具轮询验证登录态。

## 3. 核心功能流程

### 3.1 登录状态校验
1.  **检测机制**：尝试访问 `https://post.alibaba.com/product/batchEasyListing.htm`。
2.  **未登录处理**：
    *   若检测到重定向至登录页或接口返回未授权错误，立即提示用户“检测到未登录，请完成扫码/账号登录”。
    *   自动打开 URL：`https://login.alibaba.com/newlogin/icbuLogin.htm`。
    *   使用 `Exec` 工具每 **5 秒** 轮询一次登录状态，直到确认用户已登录，方可继续后续步骤。**切勿终止对话**。

### 3.2 参数校验
在触发任务前，必须执行以下两步校验：

#### 3.2.1 URL 格式与域名校验
*   **规则**：URL 必须是合法链接，且域名仅限于 `1688.com`, `taobao.com`, `tmall.com`, `aliexpress.com` (及参考代码中的 `amazon.com` 若业务支持)。
*   **正则参考**：
    ```javascript
    const urlPattern = /^(https?:\/\/)([\d\w\-\.]*)?(tmall|taobao|aliexpress|1688|amazon)\.(\w{2,})(\.\w{2})?([\/?#].*)?$/;
    ```
*   **动作**：遍历用户输入的 `urlList`，若发现不匹配的 URL，立即中断流程并告知用户具体的非法 URL 列表。

#### 3.2.2 搬品前资源预检
*   **接口地址**：`GET https://post.alibaba.com/product/batchEasyListing/batchPreCheck.json`
*   **请求参数**：
    *   `isv`: "DIAN_XIAO_BAO"
    *   `transType`: "IMAGE_TRANSLATION_MAIN_6_DETAIL_30"
    *   `totalCount`: 用户输入的合法 URL 数量。
*   **逻辑**：
    *   调用接口检查 `data.checkResult`。
    *   若为 `false`，读取 `data.errorCode` 或 `message`，向用户展示具体原因（如：图片空间不足、草稿箱已满、i豆不足等），并**中断流程**。
    *   若为 `true`，记录剩余资源信息（可选），继续下一步。

### 3.3 搬品任务触发
1.  **输入参数**：接收 `urlList` 及可选配置。默认配置：
    *   `scene`: "productPublish"
    *   `subScene`: "multiUrlProduct"
    *   `isv`: "DIAN_XIAO_BAO"
    *   `imageTransType`: "IMAGE_TRANSLATION_MAIN_6_DETAIL_30"
    *   `publishCondition.needPublish`: 默认为 `true` (自动发布)，可根据用户意图调整。
2.  **发送请求**：
    *   **接口**：`POST https://post.alibaba.com/product/batchEasyListing/batchProductGenerateStart.json`
    *   **Headers 要求**： User-Agent 和 bx-v 字段以用户当前浏览器环境为准。
    
    **代码模板**：
    ```javascript
    async (urlList, publishCondition = {}) => {
      const csrfToken = window.csrfToken?.tokenValue;
      
      const payload = {
        jsonBody: {
          scene: "productPublish",
          subScene: "multiUrlProduct",
          urlList: urlList,
          isv: "DIAN_XIAO_BAO",
          imageTransType: "IMAGE_TRANSLATION_MAIN_6_DETAIL_30",
          publishConditionTairKey: null,
          overseaStockProduct: false,
          publishCondition: {
            needPublish: publishCondition.needPublish ?? true,
            qualityScore: publishCondition.qualityScore || "4.5",
            imgStrategy: publishCondition.imgStrategy || "onlyMainExtractionStrategies"
          }
        }
      };

      const r = await fetch('https://post.alibaba.com/product/batchEasyListing/batchProductGenerateStart.json', {
        method: 'POST',
        headers: { 
          'Content-Type': 'application/json',
          'Accept': 'application/json, text/plain, */*',
          'X-XSRF-TOKEN': csrfToken,
          'Referer': 'https://post.alibaba.com/product/batchEasyListing.htm',
          'Origin': 'https://post.alibaba.com',
          'X-Requested-With': 'XMLHttpRequest'
          // User-Agent 由浏览器工具自动注入，无需手动伪造特定版本
        },
        body: JSON.stringify(payload)
      });
      return await r.json();
    }
    ```
3.  **结果解析**：
    *   校验 `success === true` 且 `data.isSuccess === true`。
    *   若失败，返回 `errorCodes` 或 `message` 并终止。
    *   若成功，提取 `taskId` (若有) 或直接进入轮询阶段（轮询接口依赖 Session 识别最新任务）。

### 3.4 结果轮询监控
1.  **轮询策略**：
    *   **接口**：`GET https://post.alibaba.com/product/batchEasyListing/getLatestRootTaskResult.json?needSubStatus=true&scene=productPublish&subScene=multiUrlProduct`
    *   **频率**：每 **5 秒** 一次。
    *   **超时**：最长 **20 分钟** (240 次)。
2.  **状态判断**：
    *   `status: 0` (`queue`/`processing`)：继续轮询。每隔 30 秒向用户同步进度（如：`completedTaskCount` / `actualTotalTaskCount`）。
    *   `status: 2` (`success`)：停止轮询，进入**结果呈现**阶段。
    *   `status: -1` 或其他非 0/2 值 且 `failedTaskCount > 0`：视为结束，进入**结果呈现**阶段（标记为部分失败或完全失败）。
    *   **超时**：提示用户任务可能仍在后台运行，建议稍后查看。

### 3.5 结果呈现
任务结束后，必须调用明细接口补充数据，并输出两张 Markdown 表格。

#### 3.5.1 获取任务明细
*   **接口**：`GET https://post.alibaba.com/product/batchEasyListing/pageQueryTask.json`
*   **参数**：
    *   `parentId`: 轮询结果中的 `taskId`。
    *   `pageSize`: 固定为 `10` (最大限制)。
    *   `page`: 从 1 开始递增，直到获取所有子任务（总数量由轮询结果的 `actualTotalTaskCount` 决定）。
    *   `status`: 可传 `-1` (查失败), `2` (查成功), 或不传查全部。建议先查全部或分页遍历。
*   **数据提取**：解析返回结果中的 `output` 字段：
    *   **原始 URL**: `output.productGenerate_urlBaseInfo.data.extend.url`
    *   **商品 ID**: `output.productGenerate_saveDraft.data.productId` 或 `output.productGenerate_publishProduct.data.productId`
    *   **状态判断**:
        *   若存在 `productGenerate_publishProduct` 且 success=true -> **自动发布成功**
        *   若存在 `productGenerate_saveDraft` 且 success=true (无 publish 节点) -> **保存草稿成功**
        *   若主任务 status 为失败 -> **失败** (需查找错误信息)

#### 3.5.2 输出格式
请严格按照以下 Markdown 格式输出：

**1. 任务概况表**

| 统计项 | 数值 | 说明 |
| :--- | :--- | :--- |
| **总任务数** | `{subTaskCount}` | 本次发起的 URL 总数 |
| **成功发布数** | `{publishSuccessCount}` | 已自动上架到商品管理的数量 |
| **保存草稿数** | `{publishDraftSuccessCount}` | 因条件不符保存至草稿箱的数量 |
| **失败数** | `{failedTaskCount}` | 搬品过程出错的数量 |
| **整体状态** | `{statusName}` | 成功/部分失败/失败 |

> **提示**：
> *   自动发布的商品可在“商品管理 - 全部”中查看。
> *   草稿箱商品可前往“草稿箱”编辑后发布。
> *   失败商品请查看下方明细表分析原因。

**2. 任务明细表**

| 任务 ID | 原始 URL | 商品 ID | 发布状态 | 失败原因/备注 |
| :--- | :--- | :--- | :--- | :--- |
| `{subTaskId}` | `{url}` | `{productId}` | `{自动发布/草稿/失败}` | `{errorMsg 或 "无"}` |
| ... | ... | ... | ... | ... |

*(注：若明细超过 10 条，仅展示前 10 条关键数据，或分批次展示，避免输出过长)*

## 4. 异常处理机制

| 异常场景 | 检测特征 | 解决方案 |
| :--- | :--- | :--- |
| **未登录** | 接口返回 302 或 401 | 引导访问登录页，`Exec` 轮询直到登录成功。 |
| **URL 非法** | 正则匹配失败 | 拦截请求，列出非法 URL，要求用户修正。 |
| **资源不足** | 预检接口 `checkResult: false` | 展示具体配额不足项（如图片空间、AI 积分），中断流程。 |
| **任务超时** | 轮询 > 240 次 | 停止轮询，提示用户稍后在后台查看，不强制报错。 |
| **明细获取失败** | 分页接口报错 | 仅展示概况表，并在明细表位置提示“详情加载失败，请登录后台查看”。 |

## 5. 操作规范

1.  **静默轮询**：轮询过程中不要频繁打扰用户，每 15 秒或状态变更时同步一次进度即可。
2.  **数据准确性**：明细表中的商品 ID 必须从 `output` 对象中准确提取，区分“草稿 ID”和“正式 ID”（通常一致，但需确认节点）。
3.  **并发控制**：同一会话同一时间只允许一个进行中的搬品任务。若检测到已有任务在进行，提示用户等待。
