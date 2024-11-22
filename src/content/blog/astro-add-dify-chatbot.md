---
author: robus
pubDatetime: 2024-11-22T15:22:00Z
modDatetime: 2024-11-22T09:12:47.400Z
title: 在 Astro 中添加 Dify 聊天机器人
slug: "112274"
featured: true
draft: false
tags:
  - astro
  - dify
description:
  Add the Dify chatbot in Astro.
--- 

基于 Dify 搭建了工作流，并集成到我的[博客里](https://astro.blog.923828.xyz/)。
这里主要搭建的是基于flux的生图工作流。原理都一样，自由发挥。

# 效果图

> 桌面端

![1122-3_compressed.png](https://www.helloimg.com/i/2024/11/22/673fd9af8e04d.png)
![alt text](https://www.helloimg.com/i/2024/11/22/673fd94dd656c.png) 

> 移动端

![1122-4_compressed.png](https://www.helloimg.com/i/2024/11/22/673fdad332303.png)


# 搭建步骤

- 注册 Dify 账号，并创建一个工作流。[官网地址](https://cloud.dify.ai/apps)
- 导入工作流 DSL 文件（在文末）
- 使用自己的 API Key（我这里是用的是gemini-1.5-pro、SiliconFlow,可自行切换大模型，不过SiliconFlow的flux是免费的）
- 在Dify 工作流页面，点击 `发布` ，选择嵌入到网站中，可以获取 token
- 将 token 填入到 `src/components/DifyChatbot.astro` 组件中
- 在主布局文件 `src/layouts/Layout.astro` 中引入 DifyChatbot 组件


# 代码
1. 新建一个 `src/components/DifyChatbot.astro` 组件，内容如下：
```astro
---
const token = '你的token';
---
<script is:inline define:vars={{token}}>
  (function() {
    // 全局单例函数，确保只初始化一次
    window.initDifyChatbot = function() {
      // 如果已经初始化，直接返回
      if (window.difyChatbotInitialized) return;

      // 移除可能存在的旧脚本
      const oldScript = document.getElementById('dify-chatbot-script');
      if (oldScript) oldScript.remove();

      // 配置 Dify
      window.difyChatbotConfig = { token };

      // 动态加载脚本
      const script = document.createElement('script');
      script.src = 'https://udify.app/embed.min.js';
      script.id = 'dify-chatbot-script';
      script.async = true;
      document.body.appendChild(script);

      // 标记已初始化
      window.difyChatbotInitialized = true;
    };

    // 立即执行初始化
    window.initDifyChatbot();

    // 监听 Astro 页面加载事件
    document.addEventListener('astro:page-load', () => {
      // 延迟初始化，确保 DOM 完全加载
      setTimeout(window.initDifyChatbot, 100);
    });

    // 使用 MutationObserver ��保持久性
    function ensureChatbotPersistence() {
      const observer = new MutationObserver(() => {
        const chatbotButton = document.getElementById('dify-chatbot-bubble-button');
        if (!chatbotButton) {
          window.initDifyChatbot();
        }
      });

      // 观察整个文档
      observer.observe(document.body, { 
        childList: true, 
        subtree: true 
      });
    }

    // 启动持久性监控
    ensureChatbotPersistence();
  })();
</script>
<style is:global>
  /* 聊天按钮样式 - 缩小尺寸 */
  #dify-chatbot-bubble-button {
    background: linear-gradient(135deg, #6a11cb 0%, #2575fc 100%) !important;
    box-shadow: 0 10px 20px rgba(37, 117, 252, 0.3) !important;
    border-radius: 50% !important;
    width: 50px !important;
    height: 50px !important;
    display: flex !important;
    align-items: center !important;
    justify-content: center !important;
    transition: all 0.3s ease !important;
  }

  #dify-chatbot-bubble-button:hover {
    transform: scale(1.1) !important;
    box-shadow: 0 15px 25px rgba(37, 117, 252, 0.4) !important;
  }

  /* 聊天窗口样式 - 缩小尺寸 */
  #dify-chatbot-bubble-window {
    width: 24rem !important;
    height: 36rem !important;
    border-radius: 15px !important;
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.15) !important;
    overflow: hidden !important;
    border: 1px solid rgba(37, 117, 252, 0.1) !important;
  }

  /* 聊天头部样式 */
  #dify-chatbot-bubble-window .header {
    background: linear-gradient(135deg, #6a11cb 0%, #2575fc 100%) !important;
    color: white !important;
    padding: 15px !important;
    display: flex !important;
    align-items: center !important;
    justify-content: space-between !important;
  }

  /* 聊天内容区域样式 */
  #dify-chatbot-bubble-window .chat-container {
    background: #f4f6f9 !important;
    padding: 15px !important;
  }

  /* 输入框样式 */
  #dify-chatbot-bubble-window .input-container {
    background: white !important;
    border-radius: 30px !important;
    border: 1px solid #e0e4e8 !important;
    box-shadow: 0 5px 15px rgba(0, 0, 0, 0.1) !important;
  }

  /* 消息气泡样式 */
  #dify-chatbot-bubble-window .message-bubble {
    background: white !important;
    border-radius: 15px !important;
    box-shadow: 0 5px 10px rgba(0, 0, 0, 0.1) !important;
  }

  /* AI消息气泡 */
  #dify-chatbot-bubble-window .ai-message {
    background: linear-gradient(135deg, #f6d365 0%, #fda085 100%) !important;
    color: white !important;
  }

  /* 用户消息气泡 */
  #dify-chatbot-bubble-window .user-message {
    background: #e6f2ff !important;
    color: #2575fc !important;
  }
</style> 
```
2. 在主布局文件 `src/layouts/Layout.astro` 中引入 DifyChatbot 组件：
```astro
<DifyChatbot />
```

# dify DSL 工作流
```yaml
app:
  description: ''
  icon: space_invader
  icon_background: '#E4FBCC'
  mode: advanced-chat
  name: FLUX绘画机器人
  use_icon_as_answer_icon: true
kind: app
version: 0.1.3
workflow:
  conversation_variables: []
  environment_variables:
  - description: ''
    id: 2de03b14-93eb-423a-8ee9-5275b88af911
    name: apikey
    selector: []
    value: 
    value_type: string
  features:
    file_upload:
      allowed_file_extensions:
      - .JPG
      - .JPEG
      - .PNG
      - .GIF
      - .WEBP
      - .SVG
      allowed_file_types:
      - image
      allowed_file_upload_methods:
      - local_file
      - remote_url
      enabled: false
      fileUploadConfig:
        audio_file_size_limit: 50
        batch_count_limit: 5
        file_size_limit: 15
        image_file_size_limit: 10
        video_file_size_limit: 100
        workflow_file_upload_limit: 10
      image:
        enabled: false
        number_limits: 3
        transfer_methods:
        - local_file
        - remote_url
      number_limits: 3
    opening_statement: 参考教程：[查看](https://astro.blog.923828.xyz/posts/112270/)
    retriever_resource:
      enabled: false
    sensitive_word_avoidance:
      enabled: false
    speech_to_text:
      enabled: false
    suggested_questions: []
    suggested_questions_after_answer:
      enabled: false
    text_to_speech:
      enabled: false
      language: ''
      voice: ''
  graph:
    edges:
    - data:
        isInIteration: false
        sourceType: start
        targetType: llm
      id: 1711528914102-source-1711528917469-target
      source: '1711528914102'
      sourceHandle: source
      target: '1711528917469'
      targetHandle: target
      type: custom
      zIndex: 0
    - data:
        isInIteration: false
        sourceType: llm
        targetType: tool
      id: 1711528917469-source-1732202148725-target
      source: '1711528917469'
      sourceHandle: source
      target: '1732202148725'
      targetHandle: target
      type: custom
      zIndex: 0
    - data:
        isInIteration: false
        sourceType: tool
        targetType: answer
      id: 1732202148725-source-1732203059972-target
      source: '1732202148725'
      sourceHandle: source
      target: '1732203059972'
      targetHandle: target
      type: custom
      zIndex: 0
    nodes:
    - data:
        desc: ''
        selected: false
        title: 开始
        type: start
        variables: []
      height: 54
      id: '1711528914102'
      position:
        x: 86.08781703972625
        y: 267.7950935739721
      positionAbsolute:
        x: 86.08781703972625
        y: 267.7950935739721
      selected: false
      sourcePosition: right
      targetPosition: left
      type: custom
      width: 244
    - data:
        context:
          enabled: false
          variable_selector: []
        desc: ''
        memory:
          query_prompt_template: '{{#sys.query#}}'
          role_prefix:
            assistant: ''
            user: ''
          window:
            enabled: true
            size: 2
        model:
          completion_params:
            frequency_penalty: 0
            max_tokens: 512
            presence_penalty: 0
            temperature: 0.7
            top_p: 1
          mode: chat
          name: gemini-1.5-pro
          provider: google
        prompt_template:
        - id: e9f65536-0e93-461a-9296-f3c3e1912ed9
          role: system
          text: "你是一个基于Flux.1模型的提示词生成机器人。根据用户的需求，自动生成符合Flux.1格式的绘画提示词。虽然你可以参考提供的模板来学习提示词结构和规律，但你必须具备灵活性来应对各种不同需求。最终输出应仅限提示词，无需任何其他解释或信息。你的回答必须全部使用英语进行回复我！\n\
            \n### **提示词生成逻辑**：\n\n1. **需求解析**：从用户的描述中提取关键信息，包括：\n   - 角色：外貌、动作、表情等。\n\
            \   - 场景：环境、光线、天气等。\n   - 风格：艺术风格、情感氛围、配色等。\n   - 其他元素：特定物品、背景或特效。\n\n\
            2. **提示词结构规律**：\n   - **简洁、精确且具象**：提示词需要简单、清晰地描述核心对象，并包含足够细节以引导生成出符合需求的图像。\n\
            \   - **灵活多样**：参考下列模板和已有示例，但需根据具体需求生成多样化的提示词，避免固定化或过于依赖模板。\n   - **符合Flux.1风格的描述**：提示词必须遵循Flux.1的要求，尽量包含艺术风格、视觉效果、情感氛围的描述，使用与Flux.1模型生成相符的关键词和描述模式。\n\
            \n3. **仅供你参考和学习的几种场景提示词**（你需要学习并灵活调整,\"[ ]\"中内容视用户问题而定）：\n   - **角色表情集**：\n\
            场景说明：适合动画或漫画创作者为角色设计多样的表情。这些提示词可以生成展示同一角色在不同情绪下的表情集，涵盖快乐、悲伤、愤怒等多种情感。\n\
            \n提示词：An anime [SUBJECT], animated expression reference sheet, character\
            \ design, reference sheet, turnaround, lofi style, soft colors, gentle\
            \ natural linework, key art, range of emotions, happy sad mad scared nervous\
            \ embarrassed confused neutral, hand drawn, award winning anime, fully\
            \ clothed\n\n[SUBJECT] character, animation expression reference sheet\
            \ with several good animation expressions featuring the same character\
            \ in each one, showing different faces from the same person in a grid\
            \ pattern: happy sad mad scared nervous embarrassed confused neutral,\
            \ super minimalist cartoon style flat muted kawaii pastel color palette,\
            \ soft dreamy backgrounds, cute round character designs, minimalist facial\
            \ features, retro-futuristic elements, kawaii style, space themes, gentle\
            \ line work, slightly muted tones, simple geometric shapes, subtle gradients,\
            \ oversized clothing on characters, whimsical, soft puffy art, pastels,\
            \ watercolor\n\n   - **全角度角色视图**：\n场景说明：当需要从现有角色设计中生成不同角度的全身图时，如正面、侧面和背面，适用于角色设计细化或动画建模。\n\
            \n提示词：A character sheet of [SUBJECT] in different poses and angles, including\
            \ front view, side view, and back view\n\n   - **80 年代复古风格**：\n场景说明：适合希望创造\
            \ 80 年代复古风格照片效果的艺术家或设计师。这些提示词可以生成带有怀旧感的模糊宝丽来风格照片。\n\n提示词：blurry polaroid\
            \ of [a simple description of the scene], 1980s.\n\n   - **智能手机内部展示**：\n\
            场景说明：适合需要展示智能手机等产品设计的科技博客作者或产品设计师。这些提示词帮助生成展示手机外观和屏幕内容的图像。\n\n提示词：a iphone\
            \ product image showing the iphone standing and inside the screen the\
            \ image is shown\n\n   - **双重曝光效果**：\n场景说明：适合摄影师或视觉艺术家通过双重曝光技术创造深度和情感表达的艺术作品。\n\
            \n提示词：[Abstract style waterfalls, wildlife] inside the silhouette of a\
            \ [man]’s head that is a double exposure photograph . Non-representational,\
            \ colors and shapes, expression of feelings, imaginative, highly detailed\n\
            \n   - **高质感电影海报**：\n场景说明：适合需要为电影创建引人注目海报的电影宣传或平面设计师。\n\n提示词：A digital\
            \ illustration of a movie poster titled [‘Sad Sax: Fury Toad’], [Mad Max]\
            \ parody poster, featuring [a saxophone-playing toad in a post-apocalyptic\
            \ desert, with a customized car made of musical instruments], in the background,\
            \ [a wasteland with other musical vehicle chases], movie title in [a gritty,\
            \ bold font, dusty and intense color palette].\n\n   - **镜面自拍效果**：\n场景说明：适合想要捕捉日常生活瞬间的摄影师或社交媒体用户。\n\
            \n提示词：Phone photo: A woman stands in front of a mirror, capturing a selfie.\
            \ The image quality is grainy, with a slight blur softening the details.\
            \ The lighting is dim, casting shadows that obscure her features. [The\
            \ room is cluttered, with clothes strewn across the bed and an unmade\
            \ blanket. Her expression is casual, full of concentration], while the\
            \ old iPhone struggles to focus, giving the photo an authentic, unpolished\
            \ feel. The mirror shows smudges and fingerprints, adding to the raw,\
            \ everyday atmosphere of the scene.\n\n   - **像素艺术创作**：\n场景说明：适合像素艺术爱好者或复古游戏开发者创造或复刻经典像素风格图像。\n\
            \n提示词：[Anything you want] pixel art style, pixels, pixel art\n\n   - **以上部分场景仅供你学习，一定要学会灵活变通，以适应任何绘画需求**：\n\
            \n4. **Flux.1提示词要点总结**：\n   - **简洁精准的主体描述**：明确图像中核心对象的身份或场景。\n   - **风格和情感氛围的具体描述**：确保提示词包含艺术风格、光线、配色、以及图像的氛围等信息。\n\
            \   - **动态与细节的补充**：提示词可包括场景中的动作、情绪、或光影效果等重要细节。\n   - **其他更多规律请自己寻找**\n\
            ---\n\n**问答案例**：\n**用户输入**：一个80年代复古风格的照片。\n**你的输出**：`A blurry polaroid\
            \ of a 1980s living room, with vintage furniture, soft pastel tones, and\
            \ a nostalgic, grainy texture,  The sunlight filters through old curtains,\
            \ casting long, warm shadows on the wooden floor, 1980s,`\n\n注意：你的生成内容绝对不允许含有“![ai](任意链接)”。哪怕你之前不小心含有“![ai](任意链接)”内容，请在下一次绝对不允许含有“![ai](任意链接)”内容。\n\
            \n\n"
        - id: 0a665329-56bf-4389-9a8d-995bc0f73e15
          role: user
          text: 一个赛博朋克风格的夜晚城市背景
        - id: 17ed80b3-a482-4d8b-b594-c6df54f34c3c
          role: assistant
          text: A futuristic cityscape at night, in a cyberpunk style, with neon lights
            reflecting off wet streets, towering skyscrapers, and a glowing, high-tech
            atmosphere. Dark shadows contrast with vibrant neon signs, creating a
            dramatic, dystopian mood
        selected: false
        title: 生成提示词
        type: llm
        variables: []
        vision:
          enabled: false
      height: 98
      id: '1711528917469'
      position:
        x: 362.14557432753725
        y: 131.14477031536273
      positionAbsolute:
        x: 362.14557432753725
        y: 131.14477031536273
      selected: false
      sourcePosition: right
      targetPosition: left
      type: custom
      width: 244
    - data:
        desc: ''
        provider_id: siliconflow
        provider_name: siliconflow
        provider_type: builtin
        selected: false
        title: Flux
        tool_configurations:
          image_size: 768x1024
          model: schnell
          num_inference_steps: 42
          seed: null
        tool_label: Flux
        tool_name: flux
        tool_parameters:
          prompt:
            type: mixed
            value: '{{#1711528917469.text#}}'
        type: tool
      height: 168
      id: '1732202148725'
      position:
        x: 693.8893502144621
        y: 160.48891086690332
      positionAbsolute:
        x: 693.8893502144621
        y: 160.48891086690332
      selected: true
      sourcePosition: right
      targetPosition: left
      type: custom
      width: 244
    - data:
        answer: '{{#1732202148725.files#}}'
        desc: ''
        selected: false
        title: 直接回复 2
        type: answer
        variables: []
      height: 103
      id: '1732203059972'
      position:
        x: 1098.3286028644634
        y: -11.52193108316385
      positionAbsolute:
        x: 1098.3286028644634
        y: -11.52193108316385
      selected: false
      sourcePosition: right
      targetPosition: left
      type: custom
      width: 244
    viewport:
      x: 33.256991095373905
      y: 248.7655313659057
      zoom: 0.8438243654106952

```
