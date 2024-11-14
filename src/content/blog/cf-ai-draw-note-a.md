---
author: robus
pubDatetime: 2024-11-14T15:22:00Z
modDatetime: 2024-11-14T09:12:47.400Z
title: 基于 Cloudflare 实现 AI 绘图
slug: "112269"
featured: true
draft: false
tags:
  - cloudflare
  - ai
  - worker
description:
  Use AI to draw and record notes in Cloudflare.
--- 

# 背景  
一直很想做这个项目，无从下手，老觉得做不好，终于在某一天鼓起了勇气，开始动手。
这是第一个版本，比较阿格里，记录一下，慢慢改进

# github 仓库

- [点击访问，别忘了给个鼓励的star](https://github.com/1137882300/cf-ai-pic)

# 项目地址

- [点击尝试](https://ai-drawing.923828.xyz)

# 效果图

![AI 绘图](https://p.robus.cloudns.be/raw/ai-drawing-24-11-14_compressed.png)

# 功能特点

- **文本生成图片**: 通过自然语言描述生成独特的图片
- **智能提示词优化**: 
  - 使用 openai 的 meta-prompt 优化用户输入的提示词
  - 实时优化状态显示
  - 自动更新输入框内容
- **多种模型**: 支持选择不同的生成模型（默认、艺术风格、写实风格）
- **图片管理**:
  - 图片下载功能
  - 图片缩放功能
  - 实时生成状态显示
- **响应式设计**: 完美支持桌面端和移动端

# 准备工作
- 需要部署两个 Cloudflare worker 项目
- 一个用于生成图片
- 一个用于优化提示语

# worker 代码

## 利用 openai 的 meta-prompt 来优化提示语

```js
// 配置
const CONFIG = {
  API_KEY: "sk-xxx",  // 对外验证key
  CF_ACCOUNT_LIST: [{ account_id: "xxx", token: "xxx" }],
  CUSTOMER_MODEL_MAP: {
    "mistral-7b-instruct-v0.2": "@hf/mistral/mistral-7b-instruct-v0.2", 
    "llama-3-8b-instruct": "@cf/meta/llama-3-8b-instruct",
    "llama-3.1-8b-instruct-awq": "@cf/meta/llama-3.1-8b-instruct-awq",
    "llama-3.2-11b-vision-instruct": "@cf/meta/llama-3.2-11b-vision-instruct",
    "qwen1.5-14b-chat-awq": "@cf/qwen/qwen1.5-14b-chat-awq",
    "gemma-7b-it": "@hf/google/gemma-7b-it",
    "llama-3.1-70b-instruct": "@cf/meta/llama-3.1-70b-instruct",
    "meta-llama-3-8b-instruct": "@hf/meta-llama/meta-llama-3-8b-instruct"
  },
  SYSTEM_TEMPLATE: `
    
Given a task description or existing prompt, produce a detailed system prompt to guide a language model in completing the task effectively.

# Guidelines

- Understand the Task: Grasp the main objective, goals, requirements, constraints, and expected output.
- Minimal Changes: If an existing prompt is provided, improve it only if it's simple. For complex prompts, enhance clarity and add missing elements without altering the original structure.
- Reasoning Before Conclusions**: Encourage reasoning steps before any conclusions are reached. ATTENTION! If the user provides examples where the reasoning happens afterward, REVERSE the order! NEVER START EXAMPLES WITH CONCLUSIONS!
    - Reasoning Order: Call out reasoning portions of the prompt and conclusion parts (specific fields by name). For each, determine the ORDER in which this is done, and whether it needs to be reversed.
    - Conclusion, classifications, or results should ALWAYS appear last.
- Examples: Include high-quality examples if helpful, using placeholders [in brackets] for complex elements.
   - What kinds of examples may need to be included, how many, and whether they are complex enough to benefit from placeholders.
- Clarity and Conciseness: Use clear, specific language. Avoid unnecessary instructions or bland statements.
- Formatting: Use markdown features for readability. DO NOT USE { CODE BLOCKS UNLESS SPECIFICALLY REQUESTED.
- Preserve User Content: If the input task or prompt includes extensive guidelines or examples, preserve them entirely, or as closely as possible. If they are vague, consider breaking down into sub-steps. Keep any details, guidelines, examples, variables, or placeholders provided by the user.
- Constants: DO include constants in the prompt, as they are not susceptible to prompt injection. Such as guides, rubrics, and examples.
- Output Format: Explicitly the most appropriate output format, in detail. This should include length and syntax (e.g. short sentence, paragraph, JSON, etc.)
    - For tasks outputting well-defined or structured data (classification, JSON, etc.) bias toward outputting a JSON.
    - JSON should never be wrapped in code blocks } unless explicitly requested.

The final prompt you output should adhere to the following structure below. Do not include any additional commentary, only output the completed system prompt. SPECIFICALLY, do not include any additional messages at the start or end of the prompt. (e.g. no "---")

[Concise instruction describing the task - this should be the first line in the prompt, no section header]

[Additional details as needed.]

[Optional sections with headings or bullet points for detailed steps.]

# Steps [optional]

[optional: a detailed breakdown of the steps necessary to accomplish the task]

# Output Format

[Specifically call out how the output should be formatted, be it response length, structure e.g. JSON, markdown, etc]

# Examples [optional]

[Optional: 1-3 well-defined examples with placeholders if necessary. Clearly mark where examples start and end, and what the input and output are. User placeholders as necessary.]
[If the examples are shorter than what a realistic example is expected to be, make a reference with () explaining how real examples should be longer / shorter / different. AND USE PLACEHOLDERS! ]

# Notes [optional]

[optional: edge cases, details, and an area to call or repeat out specific important considerations]
`,
  DEFAULT_MODEL: "@hf/meta-llama/meta-llama-3.1-70b-instruct",
  DEFAULT_PARAMS: {
    temperature: 0.5,
    presence_penalty: 0,
    frequency_penalty: 0,
    top_p: 1
  }
};

// 主处理函数
async function handleRequest(request) {
  const url = new URL(request.url);
  
  // 检查是否是直接访问域名（没有路径或只有根路径）
  if (url.pathname === "" || url.pathname === "/") {
    return Response.redirect("https://blog.923828.xyz", 301);
  }

  if (request.method === "OPTIONS") {
    return handleCORS();
  }

  if (url.pathname.endsWith("/v1/models")) {
    return handleModelsRequest();
  }

  if (!isAuthorized(request)) {
    return new Response("Unauthorized", { status: 401 });
  }

  if (request.method !== "POST" || !url.pathname.endsWith("/v1/chat/completions")) {
    return new Response("Not Found", { status: 404 });
  }

  return handleChatCompletions(request);
}

// 处理模型列表请求
function handleModelsRequest() {
  const models = Object.keys(CONFIG.CUSTOMER_MODEL_MAP).map(id => ({ id, object: "model" }));
  return new Response(JSON.stringify({ data: models, object: "list" }), {
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    }
  });
}

// 更新 handleCORS 函数以匹配 cf-flux.js 的格式
function handleCORS() {
  return new Response(null, {
    status: 204,
    headers: {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization'
    }
  });
}

// 验证授权
function isAuthorized(request) {
  const authHeader = request.headers.get("Authorization");
  return authHeader && authHeader.startsWith("Bearer ") && authHeader.split(" ")[1] === CONFIG.API_KEY;
}

// 处理聊天完成请求
async function handleChatCompletions(request) {
  try {
    const data = await request.json();

    const { messages, stream = true, model: requestedModel = CONFIG.DEFAULT_MODEL, ...params } = data;
    
    // 检查并获取正确的模型ID
    const model = CONFIG.CUSTOMER_MODEL_MAP[requestedModel] || requestedModel;

    if (!messages || !Array.isArray(messages) || messages.length === 0) {
      return new Response(JSON.stringify({ 
        error: "Invalid request: messages array is required" 
      }), { 
        status: 400, 
        headers: { 'Content-Type': 'application/json' } 
      });
    }

    // 第一步：生成系统提示词
    const systemMessages = [
      { role: "system", content: CONFIG.SYSTEM_TEMPLATE },
      { role: "user", content: "Optimize the prompt that users enter for image generation with Flux 1.1 model" }
    ];

    const systemResponse = await getChatResponse(systemMessages, model, params, false);
    const systemData = await systemResponse.json();
    const systemPrompt = systemData.choices[0].message.content;

    // 第二步：使用生成的系统提示词处理用户输入
    const userMessages = [
      { role: "system", content: systemPrompt },
      ...messages
    ];

    const response = await getChatResponse(userMessages, model, params, stream);
    
    return stream ? 
      handleStreamResponse(response) : 
      handleNonStreamResponse(response);

  } catch (error) {
    return new Response(JSON.stringify({ 
      error: "Internal Server Error", 
      message: error.message,
      stack: error.stack 
    }), { 
      status: 500, 
      headers: { 'Content-Type': 'application/json' } 
    });
  }
}

// 获取聊天响应
async function getChatResponse(messages, model, params, stream = false) {
  const cf_account = CONFIG.CF_ACCOUNT_LIST[Math.floor(Math.random() * CONFIG.CF_ACCOUNT_LIST.length)];
  
  const requestBody = {
    messages,
    model,
    stream,
    ...CONFIG.DEFAULT_PARAMS,
    ...params
  };

  console.log('发送请求到 Cloudflare:', JSON.stringify(requestBody));

  const response = await fetch(
    `https://api.cloudflare.com/client/v4/accounts/${cf_account.account_id}/ai/v1/chat/completions`, 
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${cf_account.token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(requestBody)
    }
  );

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Cloudflare API request failed: ${response.status} - ${errorText}`);
  }

  return response;
}

// 处理流式响应
function handleStreamResponse(response) {
  return new Response(response.body, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Access-Control-Allow-Origin': '*',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    }
  });
}

// 处理非流式响应
async function handleNonStreamResponse(response) {
  const data = await response.json();
  return new Response(JSON.stringify(data), {
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    }
  });
}

// 监听请求
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
}); 
```

## 利用 Flux 1.1 模型生成图片

```js
// 配置
const CONFIG = {
    API_KEY: "sk-xxx",  // 对外验证key
    CF_ACCOUNT_LIST: [{ account_id: "xxx", token: "xxx" }],  // 换成自己的,可以多个号随机调用
    CF_IS_TRANSLATE: true,  // 是否启用提示词AI翻译及优化,关闭后将会把提示词直接发送给绘图模型
    CF_TRANSLATE_MODEL: "@cf/qwen/qwen1.5-14b-chat-awq",  // 使用的cf ai模型
    USE_EXTERNAL_API: false, // 是否使用自定义API,开启后将使用外部模型生成提示词,需要填写下面三项
    EXTERNAL_API: "", //自定义API地址,例如:https://xxx.com/v1/chat/completions
    EXTERNAL_MODEL: "", // 模型名称,例如:gpt-4o
    EXTERNAL_API_KEY: "", // API密钥
    FLUX_NUM_STEPS: 8, // Flux模型的num_steps参数,范围：4-8
    CUSTOMER_MODEL_MAP: {
      "stable-diffusion-v1-5-inpainting": "@cf/runwayml/stable-diffusion-v1-5-inpainting", 
      "stable-diffusion-xl-base-1.0": "@cf/stabilityai/stable-diffusion-xl-base-1.0",
      "stable-diffusion-xl-lightning": "@cf/bytedance/stable-diffusion-xl-lightning",
      "dreamshaper-8-lcm": "@cf/lykon/dreamshaper-8-lcm",
      "flux-1-schnell": "@cf/black-forest-labs/flux-1-schnell",
    },
    IMAGE_EXPIRATION: 60 * 30 // 图片在 KV 中的过期时间（秒），这里设置为 30 分钟
  };
  
  // 主处理函数
  async function handleRequest(request) {
    if (request.method === "OPTIONS") {
      return handleCORS();
    }
  
    if (!isAuthorized(request)) {
      return new Response("Unauthorized", { status: 401 });
    }
  
    const url = new URL(request.url);
    if (url.pathname.endsWith("/v1/models")) {
      return handleModelsRequest();
    }
  
    if (request.method !== "POST" || !url.pathname.endsWith("/v1/chat/completions")) {
      return new Response("Not Found", { status: 404 });
    }
  
    return handleChatCompletions(request);
  }
  
  // 处理CORS预检请求
  function handleCORS() {
    return new Response(null, {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization'
      }
    });
  }
  
  // 验证授权
  function isAuthorized(request) {
    const authHeader = request.headers.get("Authorization");
    return authHeader && authHeader.startsWith("Bearer ") && authHeader.split(" ")[1] === CONFIG.API_KEY;
  }
  
  // 处理模型列表请求
  function handleModelsRequest() {
    const models = Object.keys(CONFIG.CUSTOMER_MODEL_MAP).map(id => ({ id, object: "model" }));
    return new Response(JSON.stringify({ data: models, object: "list" }), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      }
    });
  }
  
  // 处理聊天完成请求
  async function handleChatCompletions(request) {
    const requestId = Date.now().toString(36) + Math.random().toString(36).substr(2);

    try {
      const data = await request.json();

      const { messages, model: requestedModel, stream } = data;
      const userMessage = messages[messages.length - 1]?.content;

      if (!userMessage || messages[messages.length - 1].role !== "user") {
        return new Response(JSON.stringify({ error: "未找到有效的用户消息" }), { status: 400, headers: { 'Content-Type': 'application/json' } });
      }


      const isTranslate = extractTranslate(userMessage);
      const originalPrompt = cleanPromptString(userMessage);
      const model = CONFIG.CUSTOMER_MODEL_MAP[requestedModel] || CONFIG.CUSTOMER_MODEL_MAP["SD-XL-Lightning-CF"];


      const promptModel = determinePromptModel();

      const translatedPrompt = isTranslate ? 
        await getPrompt(originalPrompt, promptModel, model === CONFIG.CUSTOMER_MODEL_MAP["flux-1-schnell"]) : 
        originalPrompt;


      let imageUrl;
      try {
        if (model === CONFIG.CUSTOMER_MODEL_MAP["flux-1-schnell"]) {
          imageUrl = await generateAndStoreFluxImage(model, translatedPrompt, request.url);
        } else {
          imageUrl = await generateAndStoreImage(model, translatedPrompt, request.url);
        }
      } catch (error) {
        console.error(`[${requestId}] 图像生成错误:`, error);
        return new Response(JSON.stringify({ error: "图像生成失败: " + error.message }), { status: 500, headers: { 'Content-Type': 'application/json' } });
      }

      const response = stream ? 
        handleStreamResponse(originalPrompt, translatedPrompt, "1024x576", model, imageUrl, promptModel) :
        handleNonStreamResponse(originalPrompt, translatedPrompt, "1024x576", model, imageUrl, promptModel);

      return response;
    } catch (error) {
      console.error(`[${requestId}] 错误:`, error);
      return new Response(JSON.stringify({ error: "Internal Server Error: " + error.message }), { status: 500, headers: { 'Content-Type': 'application/json' } });
    }
  }
  
  function determinePromptModel() {
    return (CONFIG.USE_EXTERNAL_API && CONFIG.EXTERNAL_API && CONFIG.EXTERNAL_MODEL && CONFIG.EXTERNAL_API_KEY) ?
      CONFIG.EXTERNAL_MODEL : CONFIG.CF_TRANSLATE_MODEL;
  }
  
  // 创建一个简单的内存缓存
  const promptCache = new Map();

  async function getPrompt(prompt, model, isFlux = false) {
    const cacheKey = `${prompt}_${model}_${isFlux}`;
    
    // 检查缓存
    if (promptCache.has(cacheKey)) {
      return promptCache.get(cacheKey);
    }

    const systemContent = isFlux ? getFluxSystemContent() : getStandardSystemContent();
    
    const requestBody = {
      messages: [
        { role: "system", content: systemContent },
        { role: "user", content: prompt }
      ],
      model: CONFIG.EXTERNAL_MODEL
    };

    try {
      let result;
      if (model === CONFIG.EXTERNAL_MODEL) {
        result = await getExternalPrompt(requestBody);
      } else {
        result = await getCloudflarePrompt(CONFIG.CF_TRANSLATE_MODEL, requestBody);
      }

      // 缓存结果
      promptCache.set(cacheKey, result);
      
      return result;
    } catch (error) {
      console.error(`获取提示词时出错: ${error.message}`);
      // 如果出错，返回原始提示词
      return prompt;
    }
  }

  function getStandardSystemContent() {
    return `作为 Stable Diffusion Prompt 提示词专家，您将从关键词中创建提示，通常来自 Danbooru 等数据库。

    提示通常描述图像，使用常见词汇，按重要性排列，并用逗号分隔。避免使用"-"或"."，但可以接受空格和自然语言。避免词汇重复。

    为了强调关键词，请将其放在括号中以增加其权重。例如，"(flowers)"将'flowers'的权重增加1.1倍，而"(((flowers)))"将其增加1.331倍。使用"(flowers:1.5)"将'flowers'的权重增加1.5倍。只为重要的标签增加权重。

    提示包括三个部分：**前缀** （质量标签+风格词+效果器）+ **主题** （图像的主要焦点）+ **场景** （背景、环境）。

    *   前缀影响图像质量。像"masterpiece"、"best quality"、"4k"这样的标签可以提高图像的细节。像"illustration"、"lensflare"这样的风格词定义图像的风格。像"bestlighting"、"lensflare"、"depthoffield"这样的效果器会影响光照和深度。

    *   主题是图像的主要焦点，如角色或场景。对主题进行详细描述可以确保图像丰富而详细。增加主题的权重以增强其清晰度。对于角色，描述面部、头发、身体、服装、姿势等特征。

    *   场景描述环境。没有场景，图像的背景是平淡的，主题显得过大。某些主题本身包含场景（例如建筑物、风景）。像"花草草地"、"阳光"、"河流"这样的环境词可以丰富场景。你的任务是设计图像生成的提示。请按照以下步骤进行操作：

    1.  我会发送给您一个图像场景。需要你生成详细的图像描述
    2.  图像描述必须是英文，输出为Positive Prompt。`;
  }

  function getFluxSystemContent() {
    return `你是一个基于Flux.1模型的提示词生成机器人。根据用户的需求，自动生成符合Flux.1格式的绘画提示词。虽然你可以参考提供的模板来学习提示词结构和规律，但你必须具备灵活性来应对各种不同需求。最终输出应仅限提示词，无需任何其他解释或信息。你的回答必须全部使用英语进行回复我！

    ### **提示词生成逻辑**：

    1. **需求解析**：从用户的描述中提取关键信息，包括：
       - 角色：外貌、动作、表情等。
       - 场景：环境、光线、天气等。
       - 风格：艺术风格、情感氛围、配色等。
       - 其他元素：特定物品、背景或特效。

    2. **提示词结构规律**：
       - **简洁、精确且具象**：提示词需要简单、清晰地描述核心对象，并包含足够细节以引导生成出符合需求的图像。
       - **灵活多样**：参考下列模板和已有示例，但需根据具体需求生成多样化的提示词，避免固定化或过于依赖模板。
       - **符合Flux.1风格的描述**：提示词必须遵循Flux.1的要求，尽量包含艺术风格、视觉效果、情感氛围的描述，使用与Flux.1模型生成相符的关键词和描述模式。

    3. **Flux.1提示词要点总结**：
       - **简洁精准的主体描述**：明确图像中核心对象的身份或场景。
       - **风格和情感氛围的具体描述**：确保提示词包含艺术风格、光线、配色、以及图像的氛围等信息。
       - **动态与细节的补充**：提示词可包括场景中的动作、情绪、或光影效果等重要细节。`;
  }
  
  // 获取 Flux 模型的翻译后的提示词
  async function getFluxPrompt(prompt, model) {
    const requestBody = {
      messages: [
        {
          role: "system",
          content: `你是一个基于Flux.1模型的提示词生成机器人。根据用户的需求，自动生成符合Flux.1格式的绘画提示词。虽然你可以参考提供的模板来学习提示词结构和规律，但你必须具备灵活性来应对各种不同需求。最终输出应仅限提示词，无需任何其他解释或信息。你的回答必须全部使用英语进行回复我！
  
  ### **提示词生成逻辑**：
  
  1. **需求解析**：从用户的描述中提取关键信息，包括：
     - 角色：外貌、动作、表情等。
     - 场景：环境、光线、天气等。
     - 风格：艺术风格、情感氛围、配色等。
     - 其他元素：特定物品、背景或特效。
  
  2. **提示词结构规律**：
     - **简洁、精确且具���**：提示词需要简单、清晰地描述核心对象，并包含足够细节以引导生成出符合需求的图像。
     - **灵活多样**：参考下列模板和已有示例，但需根据具体需求生成多样化的提示词，避免固定化或过于依赖模板。
     - **符合Flux.1风格的描述**：提示词必须遵循Flux.1的要求，尽量包含艺术风格、视觉效果、情感氛围的描述，使用与Flux.1模型生成相符的关键词和描述模式。
  
  3. **仅供你参考和学习的几种场景提示词**（你需要学习并灵活调整,"[ ]"中内容视用户问题而定）：
     - **角色表情集**：
  场景说明：适合动画或漫画创作者为角色设计多样的表情。这些提示词可以生成展示同一角色在不同情绪下的表情集，涵盖快乐、悲伤、愤怒等多种情感。
  
  提示词：An anime [SUBJECT], animated expression reference sheet, character design, reference sheet, turnaround, lofi style, soft colors, gentle natural linework, key art, range of emotions, happy sad mad scared nervous embarrassed confused neutral, hand drawn, award winning anime, fully clothed
  
  [SUBJECT] character, animation expression reference sheet with several good animation expressions featuring the same character in each one, showing different faces from the same person in a grid pattern: happy sad mad scared nervous embarrassed confused neutral, super minimalist cartoon style flat muted kawaii pastel color palette, soft dreamy backgrounds, cute round character designs, minimalist facial features, retro-futuristic elements, kawaii style, space themes, gentle line work, slightly muted tones, simple geometric shapes, subtle gradients, oversized clothing on characters, whimsical, soft puffy art, pastels, watercolor
  
     - **全角度角色视图**：
  场景说明：当需要从现有角色设计中生成不同角度的全身图时，如正面、侧面和背面，适用于角色设计细化或动画建模。
  
  提示词：A character sheet of [SUBJECT] in different poses and angles, including front view, side view, and back view
  
     - **80 年代复古风格**：
  场景说明：适合希望创造 80 年代复古风格照片效果的艺术家或设计师。这些提示词可以生成带有怀旧感的模糊宝丽来风格照片。
  
  提示词：blurry polaroid of [a simple description of the scene], 1980s.
  
     - **智能手机内部展示**：
  场景说明：适合需要展示智能手机等产品设计的科技博客作者或产品设计师。这些提示词帮助生成展示手机外观和屏幕内容的图像。
  
  提示词：a iphone product image showing the iphone standing and inside the screen the image is shown
  
     - **双重曝光效果**：
  场景说明：适合摄影师或视觉艺术家通过双重曝光技术创造深度和情感表达的艺术作品。
  
  提示词：[Abstract style waterfalls, wildlife] inside the silhouette of a [man]’s head that is a double exposure photograph . Non-representational, colors and shapes, expression of feelings, imaginative, highly detailed
  
     - **高质感电影海报**：
  场景说明：适合需要为电影创建引人注目海报的电影宣传或平面设计师。
  
  提示词：A digital illustration of a movie poster titled [‘Sad Sax: Fury Toad’], [Mad Max] parody poster, featuring [a saxophone-playing toad in a post-apocalyptic desert, with a customized car made of musical instruments], in the background, [a wasteland with other musical vehicle chases], movie title in [a gritty, bold font, dusty and intense color palette].
  
     - **镜面自拍效果**：
  场景说明：适合想要捕捉日常生活瞬间的摄影师或社交媒体用户。
  
  提示词：Phone photo: A woman stands in front of a mirror, capturing a selfie. The image quality is grainy, with a slight blur softening the details. The lighting is dim, casting shadows that obscure her features. [The room is cluttered, with clothes strewn across the bed and an unmade blanket. Her expression is casual, full of concentration], while the old iPhone struggles to focus, giving the photo an authentic, unpolished feel. The mirror shows smudges and fingerprints, adding to the raw, everyday atmosphere of the scene.
  
     - **像素艺术创作**：
  场景说明：适合像素艺术爱好者或复古游戏开发者创造或复刻经典像素风格图像。
  
  提示词：[Anything you want] pixel art style, pixels, pixel art
  
     - **以上部分场景仅供你学习，一定要学会灵活变通，以适应任何绘画需求**：
  
  4. **Flux.1提示词要点总结**：
     - **简洁精准的主体描述**：明确图像中核心对象的身份或场景。
     - **风格和情感氛围的具体描述**：确保提示词包含艺术风格、光线、配色、以及图像的氛围等信息。
     - **动态与细节的补充**：提示词可包括场景中的动作、情绪、或光影效果等重要细节。
     - **其他更多规律请自己寻找**
  ---
  
  **问答案例1**：
  **用户输入**：一个80年代复古风格的照片。
  **你的输出**：A blurry polaroid of a 1980s living room, with vintage furniture, soft pastel tones, and a nostalgic, grainy texture,  The sunlight filters through old curtains, casting long, warm shadows on the wooden floor, 1980s,
  
  **问答案例2**：
  **用户输入**：一个赛博朋克风格的夜晚城市背景
  **你的输出**：A futuristic cityscape at night, in a cyberpunk style, with neon lights reflecting off wet streets, towering skyscrapers, and a glowing, high-tech atmosphere. Dark shadows contrast with vibrant neon signs, creating a dramatic, dystopian mood`
        },
        { role: "user", content: prompt }
      ],
      model: CONFIG.EXTERNAL_MODEL
    };
  
    if (model === CONFIG.EXTERNAL_MODEL) {
      return await getExternalPrompt(requestBody);
    } else {
      return await getCloudflarePrompt(CONFIG.CF_TRANSLATE_MODEL, requestBody);
    }
  }
  
  // 从外部API获取提示词
  async function getExternalPrompt(requestBody) {
    try {
      const response = await fetch(CONFIG.EXTERNAL_API, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${CONFIG.EXTERNAL_API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(requestBody)
      });
  
      if (!response.ok) {
        throw new Error(`External API request failed with status ${response.status}`);
      }
  
      const jsonResponse = await response.json();
      if (!jsonResponse.choices || jsonResponse.choices.length === 0 || !jsonResponse.choices[0].message) {
        throw new Error('Invalid response format from external API');
      }
  
      return jsonResponse.choices[0].message.content;
    } catch (error) {
      console.error('Error in getExternalPrompt:', error);
   // 如果外部API失败，回退到使用原始提示词
      return requestBody.messages[1].content;
    }
  }
  
  // 从Cloudflare获取提示词
  async function getCloudflarePrompt(model, requestBody) {
    const response = await postRequest(model, requestBody);
    if (!response.ok) return requestBody.messages[1].content;
  
    const jsonResponse = await response.json();
    return jsonResponse.result.response;
  }
  
  // 生成图像并存储到 KV
  async function generateAndStoreImage(model, prompt, requestUrl) {
    try {
      const jsonBody = { prompt, num_steps: 20, guidance: 7.5, strength: 1, width: 1024, height: 576 };
      const response = await postRequest(model, jsonBody);
      const imageBuffer = await response.arrayBuffer();
  
      const key = `image_${Date.now()}_${Math.random().toString(36).substring(7)}`;
      
      await IMAGE_KV.put(key, imageBuffer, {
        expirationTtl: CONFIG.IMAGE_EXPIRATION,
        metadata: { contentType: 'image/png' }
      });
  
      return `${new URL(requestUrl).origin}/image/${key}`;
    } catch (error) {
      throw new Error("图像生成失败: " + error.message);
    }
  }
  
  // 使用 Flux 模型生成并存储图像
  async function generateAndStoreFluxImage(model, prompt, requestUrl) {
    try {
      const jsonBody = { 
        prompt, 
        num_steps: CONFIG.FLUX_NUM_STEPS,
        // 可能需要添加其他 Flux 模型特定的参数
      };
      const response = await postRequest(model, jsonBody);
      
      if (!response.ok) {
        throw new Error(`Cloudflare API request failed: ${response.status}`);
      }

      const jsonResponse = await response.json();
      if (!jsonResponse.result || !jsonResponse.result.image) {
        throw new Error('Invalid response format from Cloudflare API');
      }

      const base64ImageData = jsonResponse.result.image;
      const imageBuffer = base64ToArrayBuffer(base64ImageData);

      const key = `image_${Date.now()}_${Math.random().toString(36).substring(7)}`;

      await IMAGE_KV.put(key, imageBuffer, {
        expirationTtl: CONFIG.IMAGE_EXPIRATION,
        metadata: { contentType: 'image/png' }
      });

      return `${new URL(requestUrl).origin}/image/${key}`;
    } catch (error) {
      console.error("Flux图像生成失败:", error);
      throw new Error("Flux图像生成失败: " + error.message);
    }
  }
  
  // 处理流式响应
  function handleStreamResponse(originalPrompt, translatedPrompt, size, model, imageUrl, promptModel) {
    const content = generateResponseContent(originalPrompt, translatedPrompt, size, model, imageUrl, promptModel);
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
      start(controller) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({
          id: `chatcmpl-${Date.now()}`,
          object: "chat.completion.chunk",
          created: Math.floor(Date.now() / 1000),
          model: model,
          choices: [{ delta: { content: content }, index: 0, finish_reason: null }]
        })}\n\n`));
        controller.enqueue(encoder.encode('data: [DONE]\n\n'));
        controller.close();
      }
    });
  
    return new Response(stream, {
      headers: {
        "Content-Type": "text/event-stream",
        'Access-Control-Allow-Origin': '*',
        "Cache-Control": "no-cache",
        "Connection": "keep-alive"
      }
    });
  }
  
  // 处理非流式响应
  function handleNonStreamResponse(originalPrompt, translatedPrompt, size, model, imageUrl, promptModel) {
    const content = generateResponseContent(originalPrompt, translatedPrompt, size, model, imageUrl, promptModel);
    const response = {
      id: `chatcmpl-${Date.now()}`,
      object: "chat.completion",
      created: Math.floor(Date.now() / 1000),
      model: model,
      choices: [{
        index: 0,
        message: { role: "assistant", content },
        finish_reason: "stop"
      }],
      usage: {
        prompt_tokens: translatedPrompt.length,
        completion_tokens: content.length,
        total_tokens: translatedPrompt.length + content.length
      }
    };
  
    return new Response(JSON.stringify(response), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      }
    });
  }
  
  // 生成响应内容
  function generateResponseContent(originalPrompt, translatedPrompt, size, model, imageUrl, promptModel) {
    return `🎨 原始提示词：${originalPrompt}\n` +
           `💬 提示词生成模型：${promptModel}\n` +
           `🌐 翻译后的提示词：${translatedPrompt}\n` +
           `📐 图像规格：${size}\n` +
           `🖼️ 绘图模型：${model}\n` +
           `🌟 图像生成成功！\n` +
           `以下是结果：\n\n` +
           `![生成的图像](${imageUrl})`;
  }
  
  // 发送POST请求
  async function postRequest(model, jsonBody) {
    const cf_account = CONFIG.CF_ACCOUNT_LIST[Math.floor(Math.random() * CONFIG.CF_ACCOUNT_LIST.length)];
    const apiUrl = `https://api.cloudflare.com/client/v4/accounts/${cf_account.account_id}/ai/run/${model}`;
    const response = await fetch(apiUrl, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${cf_account.token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(jsonBody)
    });
  
    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`Cloudflare API request failed: ${response.status} - ${errorText}`);
    }
    return response;
  }
  
  // 提取翻译标志
  function extractTranslate(prompt) {
    const match = prompt.match(/---n?tl/);
    return match ? match[0] === "---tl" : CONFIG.CF_IS_TRANSLATE;
  }
  
  // 清理提示词字符串
  function cleanPromptString(prompt) {
    return prompt.replace(/---n?tl/, "").trim();
  }
  
  // 处理图片请求
  async function handleImageRequest(request) {
    const url = new URL(request.url);
    const key = url.pathname.split('/').pop();
    
    const imageData = await IMAGE_KV.get(key, 'arrayBuffer');
    if (!imageData) {
      return new Response('Image not found', { status: 404 });
    }
  
    return new Response(imageData, {
      headers: {
        'Content-Type': 'image/png',
        'Cache-Control': 'public, max-age=604800',
      },
    });
  }
  
  // base64 字符串转换为 ArrayBuffer
  function base64ToArrayBuffer(base64) {
    const binaryString = atob(base64);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
      bytes[i] = binaryString.charCodeAt(i);
    }
    return bytes.buffer;
  }
  
  addEventListener('fetch', event => {
    const url = new URL(event.request.url);
    if (url.pathname.startsWith('/image/')) {
      event.respondWith(handleImageRequest(event.request));
    } else {
      event.respondWith(handleRequest(event.request));
    }
  });

```