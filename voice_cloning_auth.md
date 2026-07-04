# Pocket-TTS 声音克隆与 Hugging Face 权限认证指南

在 Windows 上使用 **Kyutai Pocket-TTS** 进行声音克隆时，常会遇到权限认证（Gated Repo）相关的错误。本文档详细记录了该问题的成因、表现及解决方法。

---

## 1. 问题现象与成因

### 现象一：自定义声音克隆报错
当启动 Web 服务并尝试在网页上传自定义 WAV 文件进行克隆时，后台抛出如下异常：
```python
ValueError: We could not download the weights for the model with voice cloning, but you're trying to use voice cloning.
```

### 现象二：直接检测 Hugging Face 下载报错
在未授权的环境下直接下载克隆版权重时，会报 `GatedRepoError` 异常：
```
huggingface_hub.errors.GatedRepoError: 401 Client Error.
Cannot access gated repo for url https://huggingface.co/kyutai/pocket-tts/resolve/...
```

### 成因解析
1. **声音克隆权重是受限模型 (Gated Model)**：
   * 官方的声音克隆权重（`model.safetensors`）托管在受限仓库 `kyutai/pocket-tts` 中。出于防范滥用（例如未经授权的声纹模仿）的考虑，用户必须手动登录 Hugging Face 账户并**同意服务条款**才能下载。
2. **免登录版与克隆版的静默降级逻辑**：
   * 代码在加载模型时，有一个 `try-except` 容错逻辑。如果克隆版权重因未登录而下载失败，代码会**静默降级 (Fallback)** 并自动下载免登录的公开版模型权重（`kyutai/pocket-tts-without-voice-cloning`）。
   * **内置声音**（如 `alba`、`cosette` 等）可以直接使用，因为它们的声纹嵌入（Embedding）已经提前提取好并随免登录版公开了。
   * 当用户上传**自定义音频文件**时，服务必须使用克隆版权重动态提取声纹特征，此时系统检测到当前加载的是“无克隆版”权重，就会抛出 `ValueError`。

---

## 2. 解决方案步骤

要启用声音克隆功能，必须完成以下两步配置：

### 第一步：在 Hugging Face 网页端同意许可协议
1. 登录 [Hugging Face 官网](https://huggingface.co/)。
2. 打开受限模型卡片页面：[https://huggingface.co/kyutai/pocket-tts](https://huggingface.co/kyutai/pocket-tts)。
3. 在页面上方的提示框中，勾选并点击 **Accept terms and conditions**（接受条款）以获取访问授权。

### 第二步：在本地进行终端登录授权
由于本地传统的 `huggingface-cli login` 指令已被废弃，现在需要使用 `hf` 工具链登录：

1. 打开命令行，在项目虚拟环境下运行：
   ```bash
   hf auth login
   ```
2. 按照屏幕提示选择登录方式：
   * **方式 A（推荐）：Log in with your browser**
     1. 命令行会生成一个链接（如 `https://hf.co/oauth/device`）和一个设备验证码（如 `WDAD-61HE`）。
     2. 浏览器打开该链接，输入验证码确认授权。
     3. 命令行提示 `Login successful` 即表示成功。
   * **方式 B：Paste an access token**
     1. 前往 [HF Access Tokens 页面](https://huggingface.co/settings/tokens) 生成一个 Read 权限的 Token。
     2. 复制并在终端中粘贴该 Token 即可。

---

## 3. 本地开发与测试建议

### 1. 运行本地开发版本
请避免使用 `uvx pocket-tts`，因为它会从 PyPI 仓库下载线上未修改的代码。应使用 `uv run` 运行您修改过编码问题的本地项目代码：
```bash
# 启动本地服务
uv run pocket-tts serve

# 单次音频生成命令行
uv run pocket-tts generate --text "Hello world" --voice "C:/path/to/my_voice.wav"
```

### 2. 验证授权是否生效
服务启动时，日志若能成功通过验证并不再静默降级（不会有 401 报错），即代表声音克隆权重已成功加载，您可以无缝使用自定义 WAV 克隆合成功能。
