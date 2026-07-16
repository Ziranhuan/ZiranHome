---
title: NVIDIA NIM 免费 API 申请及 CC-Switch 接入 Claude Code 完整教程
date: 2026-07-16 10:00:00
tags:
  - NVIDIA
  - Claude Code
  - CC-Switch
  - 免费API
  - AI编程
categories:
  - 教程
---

## 一、引言

Claude Code 凭借其出色的代码理解与生成能力，已成为众多开发者的首选 AI 编程助手。然而，其官方版本依赖 Anthropic 付费 API，对于个人开发者和学习者来说成本较高。

好消息是，NVIDIA 通过 [build.nvidia.com](https://build.nvidia.com) 向全球开发者提供**免费 API 调用额度**，涵盖 DeepSeek V4、GLM-5、Minimax-M3 等多个业界前沿模型。

但这里有个关键问题：**NVIDIA API 与 Claude Code 的协议不兼容**，无法直接对接。本文将提供一套完整的「申请 → 搭桥 → 配置」方案，让你零成本地用 Claude Code 调用 NVIDIA 托管的各类大模型。

### 适用场景

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 轻度使用、不想付费 | ✅ 本文方案 | NVIDIA 免费额度完全满足日常编程辅助需求 |
| 高频调用、生产环境 | ❌ 官方付费 API | 免费额度有调用频率限制，不适合生产级场景 |
| 需要 Claude 原生模型 | ❌ 官方付费 API | NVIDIA 只托管第三方模型，无法使用 Claude 原生模型 |
| 想先用后买 | ✅ 本文方案 | 先用免费额度体验，再根据需求决定是否升级付费 |

### 限制须知

- **调用频率限制**：每分钟 40 次，个人日常使用绰绰有余
- **协议差异**：NVIDIA API 采用 OpenAI 兼容协议，而 Claude Code 原生只支持 Anthropic 格式（核心难点）
- **响应速度**：免费额度在高峰期优先级较低，可能出现排队延迟

---

## 二、申请 NVIDIA API Key

### 2.1 注册 NVIDIA 账号

1. 打开浏览器访问 **[build.nvidia.com](https://build.nvidia.com)**
2. 点击右上角「**Sign In**」进入登录/注册页面
3. 输入邮箱后点击「Next」，按提示完成密码设置和邮箱验证
4. 创建 NVIDIA Cloud Account——**账户名只能使用英文或数字，不支持中文**
5. 点击右上角「Verify」，输入国内手机号（+86）完成短信验证

> 💡 **快捷方式**：直接使用 Google / Microsoft / GitHub 账号登录，跳过注册流程。

![NVIDIA Build 官网界面](/images/nvidia-build-website.png)

### 2.2 生成 API Key

1. 登录后，点击右上角头像 →「**API Keys**」，或直接访问 [build.nvidia.com/settings/api-keys](https://build.nvidia.com/settings/api-keys)
2. 点击「**Generate Key**」（或「Generate New API Key」）
3. 填写 Key 名称（便于后续管理），过期时间建议选择 **永不过期（Never Expire）**
4. 点击生成

> ⚠️ **关键提醒**：生成后 **务必立即复制并保存 API Key**！关闭弹窗后将无法再次查看完整的 Key，只能删除重建。API Key 格式通常以 `nvapi-` 开头。

![生成 API Key](/images/nvidia-generate-api-key.png)

> 📋 **免费模型列表**：[build.nvidia.com/models?filters=nimType%3Anim_type_preview](https://build.nvidia.com/models?filters=nimType%3Anim_type_preview) —— 可在此查看当前支持的免费模型及详细信息。

---

## 三、为什么不能直接接入 Claude Code？

拿到 API Key 后，很多人的第一反应是尝试直接配置：

```bash
# ❌ 这样会报错，无法正常工作
export ANTHROPIC_BASE_URL=https://integrate.api.nvidia.com/v1
export ANTHROPIC_API_KEY=nvapi-xxxxx
claude
```

**结果必然报错。** 根本原因是 **协议格式不兼容**——两者使用的是完全不同的 API 协议：

| 对比项 | Claude Code 原生协议 | NVIDIA API 协议 |
|--------|---------------------|-----------------|
| 协议标准 | Anthropic 原生 API | OpenAI 兼容 API |
| 请求体结构 | `messages` + `anthropic_version` | `messages` + `model` |
| 响应格式 | `content` 数组 + `stop_reason` | `choices` 数组 + `finish_reason` |
| 流式接口 | SSE Anthropic 格式 | SSE OpenAI 格式 |

打个比方：你拿着 Type-C 充电线去插 iPhone 的 Lightning 接口——物理上就不兼容，根本无法连接。

```
Claude Code ──(Anthropic 格式请求)──> NVIDIA API
                                      ↓ 解析失败
                                 拒绝处理，返回错误
```

解决方案是引入一个 **协议转换器**，就像一个"翻译官"——它接收 Claude Code 的 Anthropic 格式请求，将其转换为 NVIDIA API 能理解的 OpenAI 格式，再将响应转换回 Anthropic 格式返回给 Claude Code。

这个"翻译官"就是 **CC-Switch**。

---

## 四、安装 CC-Switch

[CC-Switch](https://github.com/farion1231/CC-Switch) 是一款开源的 AI 编程助手配置管理工具，其核心价值在于**内置协议转换层**——它能将 Claude Code 发出的 Anthropic 格式请求自动转换为 OpenAI 格式，无缝转发到 NVIDIA 等第三方 API。

### 4.1 系统要求

- **Node.js**：CC-Switch 是 GUI 应用，但 Claude Code 依赖 Node.js 运行时。如果你尚未安装，请访问 [nodejs.org](https://nodejs.org) 下载并安装 LTS 版本。
- **操作系统**：支持 Windows / macOS / Linux

### 4.2 各平台安装

CC-Switch 的发行版托管在 GitHub Releases，访问以下地址下载对应平台的安装包：
**<https://github.com/farion1231/CC-Switch/releases>**

**Windows：**

下载 `CC-Switch-v{版本号}-Windows.msi` 安装包进行安装；或下载 `.zip` 便携版，解压后直接运行。

**macOS（推荐）：**

通过 Homebrew 一键安装：

```bash
brew tap farion1231/ccswitch
brew install --cask cc-switch
```

**Linux（Debian/Ubuntu）：**

```bash
sudo dpkg -i cc-switch_x.x.x_amd64.deb
```

安装完成后启动 CC-Switch，你将看到主界面。

> 📖 **更多安装选项**：访问 [CC-Switch GitHub 仓库](https://github.com/farion1231/CC-Switch/tree/main)，在 README 中查看详细安装说明、各版本差异及手动构建指南。

---

## 五、配置 CC-Switch 接入 NVIDIA API

### 5.1 开启本地路由

启动 CC-Switch 后，找到 **本地路由（Local Router）** 选项，将开关打开。这一步会在本地启动一个协议转换代理，默认监听端口通常是 `3001`。

> 🔧 **工作原理**：本地路由在你的电脑上启动一个 HTTP 代理服务，它接收 Claude Code 发出的 Anthropic 格式请求，将其转换为 OpenAI 格式后转发到 NVIDIA API，再将 NVIDIA 的响应转换回 Anthropic 格式返回给 Claude Code。整个过程对用户完全透明。

![开启本地路由](/images/ccswitch-local-router.png)

### 5.2 添加 NVIDIA 供应商

1. 在主界面点击 **「+ Add Provider」**（添加供应商）
2. 在提供商列表中，搜索并选择 **NVIDIA** 或 **NVIDIA NIM** 的预设模板——CC-Switch 内置了大量常用提供商的配置模板，选择预设可以免去手动填写复杂 API 地址的麻烦
3. 在 **API Key** 字段中，粘贴你之前保存的以 `nvapi-` 开头的 NVIDIA API Key
4. 在 **Model（模型）** 字段中，填入你想使用的模型 ID。模型 ID 可以在 NVIDIA 的模型页面找到，推荐以下几个热门模型：

| 模型 | 模型 ID | 特点 |
|------|---------|------|
| GLM-5 | `z-ai/glm5.2` | 中文能力强，代码生成优秀 |
| Minimax-M3 | `minimaxai/minimax-m3` | 多模态支持，推理能力出色 |
| DeepSeek V4 Flash | `deepseek-ai/deepseek-v4-flash` | 速度快，适合日常编程 |

5. 保存配置

![添加 NVIDIA 供应商](/images/ccswitch-add-provider.png)

### 5.3 激活配置

在 CC-Switch 主界面中，选中你刚刚创建的 NVIDIA 配置，点击 **「Use」** 或 **「Switch」** 按钮，使其成为当前激活的配置。

![激活配置](/images/ccswitch-activate.png)

### 5.4 测试代理

通过 curl 发送测试请求，验证代理是否正常工作：

```bash
curl -N -X POST http://localhost:3001/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: test" \
  -d '{
    "model": "z-ai/glm5.2",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Say hello in one sentence"}],
    "stream": true
  }'
```

如果能快速持续返回多行流式响应（SSE 格式），说明代理启动成功。

### 5.5 启动 Claude Code

配置完成后，在终端中直接启动：

```bash
claude
```

CC-Switch 会自动将 Claude Code 的请求路由到你配置的 NVIDIA API 上。

> 💡 **手动配置环境变量**（当自动路由失效时）：
>
> ```bash
> export ANTHROPIC_AUTH_TOKEN="freecc"
> export ANTHROPIC_BASE_URL="http://localhost:8082"
> claude
> ```

---

## 六、验证与使用

### 6.1 验证配置

启动 Claude Code 后，在界面中输入 `/models` 命令查看可用模型列表。如果能看到你配置的 NVIDIA 模型（如 `nvidia_nim/z-ai/glm5.2`），说明配置成功。

### 6.2 切换模型

在 Claude Code 中通过 `/model` 命令可以随时切换不同的 NVIDIA 模型，无需重启或修改配置。

### 6.3 常见问题

**Q：报错提示不支持某个参数（如 `xhigh`）怎么办？**

NVIDIA API 可能不支持 Claude Code 默认传递的某些参数（如 `reasoning_effort`）。解决方法是在 CC-Switch 的配置文件中调整或移除对应参数，或更换为支持该参数的模型。

**Q：API Key 找不到了怎么办？**

如果在生成后没有保存，只能前往 build.nvidia.com 的 API Keys 页面删除旧 Key，并重新生成一个新的。

**Q：响应速度很慢？**

免费额度在高峰期的优先级较低，建议避开北美白天的使用高峰（通常是北京时间凌晨）。如果追求稳定速度，可以考虑升级到 NVIDIA 的付费套餐。

**Q：每分钟 40 次限制够用吗？**

对于个人日常的编程辅助来说完全足够。如果是团队共享使用或自动化流水线场景，建议升级付费套餐以获得更高的调用频率限制。

---

## 七、总结

整体流程可以概括为简洁的三步：

1. **申请**：在 [build.nvidia.com](https://build.nvidia.com) 注册账号，生成 `nvapi-` 开头的 API Key
2. **搭桥**：安装 CC-Switch，开启本地路由，添加 NVIDIA 供应商并配置 API Key 和模型 ID
3. **使用**：启动 Claude Code，所有请求自动通过 CC-Switch 转发到 NVIDIA API

整个方案的核心在于 **协议转换**——NVIDIA API 使用 OpenAI 兼容协议，而 Claude Code 原生只支持 Anthropic 格式，CC-Switch 作为中间层完美解决了这个兼容性问题。

CC-Switch 让整个过程变得几乎无感——一次配置，永久使用。对于想要低成本体验 Claude Code + 前沿大模型的开发者来说，这是一个非常实用且值得尝试的方案。

---

### 延伸阅读

- [NVIDIA Build 平台](https://build.nvidia.com) —— 申请 API Key 和查看免费模型
- [CC-Switch GitHub 仓库](https://github.com/farion1231/CC-Switch/tree/main) —— 项目源码和详细文档
- [CC-Switch 下载页面](https://github.com/farion1231/CC-Switch/releases) —— 各平台安装包
- [NVIDIA 免费模型列表](https://build.nvidia.com/models?filters=nimType%3Anim_type_preview) —— 当前支持的所有免费模型
- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code/overview) —— Claude Code 使用指南