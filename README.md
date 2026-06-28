# Literature Digest Cloud Push

这个目录用于把“每日文献搜索 + 去重 + 企业微信推送”放到 GitHub Actions 云端运行。电脑关机也不影响推送。

## 工作方式

- GitHub Actions 每 30 分钟唤醒一次。
- 脚本按 `TIME_ZONE` 和 `PUSH_TIME` 判断是否到了当天推送时间。
- 当天已推送过则跳过。
- 到点后检索 Crossref、OpenAlex、PubMed、Europe PMC、arXiv。
- 排除已推送过的 DOI/标题。
- 可要求文献同时命中多组关键词，例如“手术相关”和“agent 相关”。
- 可优先推送指定期刊，例如 `npj Digital Medicine`、`Nature` 及 Nature 子刊。
- 按优先期刊、关键词相关度和期刊指标从高到低排序。
- 推送成功后，把已推送历史保存到 GitHub Actions cache。

## 必填配置

在 GitHub 仓库里打开：

`Settings -> Secrets and variables -> Actions`

添加以下配置。

### Secrets

`WECOM_WEBHOOK_URL`

企业微信群机器人 Webhook，例如：

```text
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=...
```

### Variables 或 Secrets

`RESEARCH_QUERY`

研究方向关键词，例如：

```text
dental implant planning CBCT deep learning
```

## 可选配置

这些可以放在 Variables，也可以放在 Secrets。

```text
PUSH_TIME=08:30
TIME_ZONE=America/New_York
LOOKBACK_DAYS=90
MAX_ITEMS=3
EXCLUDE_PUSHED=true
AUTO_JOURNAL_METRICS=true
REQUIRED_KEYWORD_GROUPS=surgery|surgical|operative|operation|procedure|intervention|intraoperative|perioperative|operating room|surgeon;agent|agents|agentic|AI agent|LLM agent|autonomous agent|multi-agent|multiagent|large language model|LLM
STRICT_REQUIRED_KEYWORDS=true
PRIORITY_JOURNALS=npj Digital Medicine,Nature
```

`OPENALEX_API_KEY` 可选。OpenAlex API Key 免费，可提高 OpenAlex 请求额度。

`IMPACT_FACTOR_TABLE` 可选，用于手动覆盖期刊指标，每行一个期刊：

```text
Journal of Dental Research=8.9
Clinical Oral Implants Research=4.8
Dentomaxillofacial Radiology=3.1
```

没有手动表时，脚本会自动从 OpenAlex 获取开放期刊指标，优先使用 `2-year mean citedness`。

## 手动测试

进入 GitHub 仓库：

`Actions -> Literature Digest Daily WeCom Push -> Run workflow`

勾选 `force` 后运行。这样会忽略当天时间判断，立即搜索并推送。

## 注意

GitHub Actions 定时任务不是秒级定时，可能有几分钟延迟。这个 workflow 每 30 分钟检查一次，所以通常会在设定时间之后的下一次运行中推送。

企业微信 Webhook 相当于密钥，不要提交到代码里，只放在 GitHub Secrets。
