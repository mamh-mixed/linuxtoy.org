Title: 基于 Koboldcpp 的浏览器 AI 聊天机器人
Date: 2026-04-11 17:00
Category: Tutorial
Tags: fedora, llm
Slug: local-browser-assistant-with-koboldcpp
Authors: lovenemesis

本系列教程的等待期超出了作者的预期。本次，我们将以 Fedora 43 为例，搭建一个基于本地运行的大语言模型工具，配合 Firefox 的 AI 聊天机器人，在没有隐私和流量顾虑的情况下，畅游互联网。本文介绍的思路和基本步骤同样适用于 OSX 和 Windows 系统。

## Koboldcpp：多用途的本地 LLM 运行环境

[Koboldcpp](https://github.com/LostRuins/koboldcpp) 在[本站之前介绍过](https://linuxtoy.org/archives/local-research-assistant-with-koboldcpp-pot-and-gpt4all.html) ，经过一年的发展，Koboldcpp 新增了以下功能：

* 无缝兼容 Claude Desktop 的 **MCP 配置**文件
* 包含 Jinja 支持的完善**通用工具调用**模式
* **内建 lcpp 轻量级 WebUI 界面**
* 进一步优化了**兼容 OpenAI 和 Ollama 接口服务**

更多详情请参考项目的[ Wiki 说明文件](https://github.com/LostRuins/koboldcpp/wiki)。Koboldcpp 提供了适用于多种平台的预编译版本，兼容无 AVX 的老旧 CPU、新旧版本的 CUDA、Mac Metal 等各种环境。考虑到更广泛的平台适用性，下面以 Vulkan 后端的版本作为示例：

1. 前往[项目发布页面](https://github.com/LostRuins/koboldcpp/releases)，下载最新发布版本 `Assets` 中的 `koboldcpp-linux-x64-nocuda`（Linux X86_64）。由于功能增多，本文撰写时最新的不包含 CUDA 运行时环境的版本体积已增加到约 115MB。
2. 将下载好的文件移动到您认为合适的位置：
```
mv -v koboldcpp-linux-x64-nocuda $HOME/bin/koboldcpp
```
3. 赋予其可执行权限：
```
chmod +x $HOME/bin/koboldcpp
```

## 模型文件：Gemma 4

[Firefox 的 AI 聊天机器人](https://support.mozilla.org/1/firefox/150.0/Linux/zh-CN/ai-chatbot) 可以为当前浏览的网页生成摘要、对选中的文本进行摘要生成、解释说明，甚至执行任意操作。但这些功能对其背后的大语言模型的工具调用能力和提示词格式都有一定的要求，常见的本地部署的 DeepSeek 和 Qwen3.5 模型往往无法满足。直到本月初，Google DeepMind 发布了[新一代的 Gemma4 模型](https://deepmind.google/models/gemma/gemma-4/)，才实现了与 Firefox AI 聊天机器人的完美兼容。Gemma4 也针对消费级设备进行了优化，其最小版本在搭载 AMD Ryzen 5600U 的轻薄本上也能快速运行：

* [google_gemma-4-E2B-it](https://www.modelscope.cn/models/bartowski/google_gemma-4-E2B-it-GGUF/resolve/master/google_gemma-4-E2B-it-Q5_K_M.gguf) 及其[用于图像视频理解的映射文件](https://www.modelscope.cn/models/bartowski/google_gemma-4-E2B-it-GGUF/resolve/master/mmproj-google_gemma-4-E2B-it-f16.gguf) ：最小版本，适用于仅有 16G 内存的设备
* [google_gemma-4-E4B-it](https://www.modelscope.cn/models/bartowski/google_gemma-4-E4B-it-GGUF/resolve/master/google_gemma-4-E4B-it-Q5_K_M.gguf) 及其[用于图像视频理解的映射文件](https://www.modelscope.cn/models/bartowski/google_gemma-4-E4B-it-GGUF/resolve/master/mmproj-google_gemma-4-E4B-it-f16.gguf) ：中等尺寸版本，适用于具备 32G 内存的设备
* [google_gemma-4-26B-A4B-it](https://www.modelscope.cn/models/bartowski/google_gemma-4-26B-A4B-it-GGUF/resolve/master/google_gemma-4-26B-A4B-it-Q5_K_M.gguf) 及其[用于图像视频理解的映射文件](https://www.modelscope.cn/models/bartowski/google_gemma-4-26B-A4B-it-Q5_K_M.gguf) ：体积较大但参数利用率最高的 MoE 版本，适用于具备至少 12G 独立显存的设备

根据自己的设备配置，从上述选择一个合适的 Gemma4 版本。在平衡体积和质量方面，笔者仍然推荐 `Q5_K_M` 的量化版本。模型文件较大，下载完成需要一定时间，之后将其移动到您认为合适的位置，例如 `$HOME/gguf`。

## 使用 Koboldcpp 命令行方式运行模型文件

接下来，我们直接通过命令行的方式将 Koboldcpp 的模型启动和配置整合成一键运行脚本。

[除了之前介绍过的参数](https://linuxtoy.org/archives/local-research-assistant-with-koboldcpp-pot-and-gpt4all.html)之外，针对 Gemma4 有一些额外需求：

* `--mmproj`：指定映射模型文件名
* `--jinja`：指定 Jinja 格式的输出模板，用于规范输出内容
* `--useswa`：开启 "Sliding Window Attention"，有效降低内存需求
* `--usemmap`：可选，使用内存映射文件，对于 MoE 场景和内存有限的场景，加载速度更快，但会影响性能

因此，对于 `gemma-4-E2B-it` 命令版本，启动方式如下：

```
$HOME/bin/koboldcpp \
	--model $HOME/gguf/google_gemma-4-E2B-it-Q5_K_M.gguf \
	--mmproj $HOME/gguf/mmproj-google_gemma-4-E2B-it-f16.gguf \
	--usevulkan \
	--gpulayers -1 \
	--skiplauncher \
	--quiet \
	--contextsize 16384 \
	--defaultgenamt 4096 \
	--jinja \
	--useswa \
	--usemmap
```

将上述命令保存到您偏好的终端脚本或批处理文件中即可。后续运行该脚本或批处理文件时，Koboldcpp 将在后台以进程方式运行，并且不会在浏览器中打开 Kobold Lite，但会在 `http://localhost:5001` 上提供 KoboldAI、OpenAI 和 Ollama 三种风格的 API 以及一个**轻量级的 WebUI**。如需退出，只需关闭终端窗口即可。

当然可以进一步利用 `systemd` 的方式将其彻底服务化并纳入用户的登录进程管理，这并不会影响外部应用的访问，所以此处不再赘述。

## Firefox AI 聊天机器人：网页办公好助手

接下来我们来看看如何将配置好的 LLM 接入到 Firefox AI 聊天机器人中。Firefox AI 聊天机器人默认仅支持少数联网服务，我们需要在它的配置界面中添加本地访问方式，指向 Koboldcpp 内置的轻量级 WebUI。

1. 在 Firefox 地址栏输入 `about:config` 打开详细配置界面
2. 了解风险后，搜索 `browser.ml.chat.hideLocalhost` 并将其设置为 `false`
3. 搜索并修改 `browser.ml.chat.provider`，指向轻量级 WebUI `http://localhost:5001/lcpp/`
4. 打开运行配置好的 Koboldcpp 终端脚本
5. 使用快捷键`CTRL + ALT + X` 开启侧边栏的 AI 聊天机器人，或者在页面空白处右键选择“询问 AI 聊天机器人”，您就能看到 lcpp 风格的轻量级 WebUI 了。也可以通过选中文字后的浮动按钮唤起。更多说明请参考 [Mozilla 的帮助文档](https://support.mozilla.org/1/firefox/150.0/Linux/zh-CN/ai-chatbot)

看到这里，一些读者可能会好奇使用常见的 `ollama` 等工具是否也能达到同样的目的？
答案是肯定的，但需要额外搭建诸如 openWebUI 之类的网页前端。因为 Firefox AI 聊天机器人不提供自己的 UI，它依赖于服务提供方的前端来实现交互。恰好 Koboldcpp 内建的轻量级 WebUI 满足了这一需求。

[参考内容](https://code.mendhak.com/firefox-local-chatbot-ollama/#firefox-config)
