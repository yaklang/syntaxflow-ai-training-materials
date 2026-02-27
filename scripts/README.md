# scripts：构建与维护脚本

本目录包含 SyntaxFlow AIKB 的构建、打包、OSS 上传和在线验证脚本。

## 脚本说明

| 脚本 | 功能 |
|------|------|
| build-syntaxflow-aikb-rag.yak | 从 basic-syntax、practice、rule-examples 的 .md 构建 RAG 索引 |
| merge-in-one-text.yak | 将文档和规则打包为 ZIP（用于训练或增量更新） |
| update-rag.yak | 基于差异 ZIP 包增量更新 RAG 知识库 |
| build-syntaxflow-aikb-to-libs.sh / .cmd | 构建 ZIP 与 RAG 到本地 libs 目录（thirdparty 安装路径） |
| check-aikb-online.yak | 检查 OSS 上的 syntaxflow-aikb 文件状态（ZIP、时间戳、RAG） |

## 使用方式

### 构建 RAG 索引

```bash
cd syntaxflow-ai-training-materials
yak scripts/build-syntaxflow-aikb-rag.yak --base . --output syntaxflow-aikb.rag
```

可选参数：`--force` 强制重建、`--concurrency N`、`--ai-type`、`--ai-model`。

### 打包 ZIP

```bash
cd syntaxflow-ai-training-materials
yak scripts/merge-in-one-text.yak --base . --output syntaxflow-aikb.zip
```

目标目录：basic-syntax、practice、rule-examples、examples-rule；扩展名：.md、.sf。

### 增量更新 RAG

```bash
yak scripts/update-rag.yak --rag-file old.rag --diff-zip syntaxflow-aikb.zip --output new.rag
```

`--diff-zip` 中应仅包含变更的 .md 文件（可通过对比新旧 merge ZIP 得到）。

### 本地构建到 libs（thirdparty 安装路径）

```bash
# 从 syntaxflow-ai-training-materials 根目录执行
./scripts/build-syntaxflow-aikb-to-libs.sh   # Linux/macOS
scripts\build-syntaxflow-aikb-to-libs.cmd    # Windows
```

输出路径：`${YAKIT_HOME}/projects/libs/`（syntaxflow-aikb.zip、syntaxflow-aikb.rag）

### 检查 OSS 在线状态

```bash
yak scripts/check-aikb-online.yak
```

检查 `https://oss-qn.yaklang.com/ai-knowledge-base/` 下的 syntaxflow-aikb.zip、syntaxflow-aikb.updated_at.txt、rag/syntaxflow-aikb.rag。

## OSS 自动上传（GitHub Actions）

当 `main` 分支指定路径有变更时，`.github/workflows/upload-aikb.yml` 会自动：

1. 构建 syntaxflow-aikb.zip
2. 构建 syntaxflow-aikb.rag（需配置 AI 相关 Secrets 时更稳定）
3. 上传到阿里云 OSS：`/ai-knowledge-base/syntaxflow-aikb.zip`、`/ai-knowledge-base/rag/syntaxflow-aikb.rag`
4. 写入时间戳文件 syntaxflow-aikb.updated_at.txt

**所需 GitHub Secrets**：`OSS_KEY_ID`、`OSS_KEY_SECRET`（上传凭证）。可选：`AI_API_KEY`、`AI_API_MODEL`、`AI_API_DOMAIN`（RAG 构建时使用）。
