{
  "name": "2",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "triggerAtHour": 9,
              "triggerAtMinute": 1
            }
          ]
        }
      },
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.4,
      "position": [
        -208,
        -16
      ],
      "id": "0efa5fa1-e6a8-43e7-a6fd-87532148cdad",
      "name": "Schedule Trigger"
    },
    {
      "parameters": {
        "url": "=https://arctic-shift.photon-reddit.com/api/posts/search?subreddit={{ ['smallbusiness','smallbusiness','smallbusiness','smallbusiness','agency','smallbusiness','smallbusiness'][$now.weekday - 1] }}&query={{ ['still+do+manually','chasing+invoices','hours+every+week','data+entry','repetitive+task','still+do+manually','chasing+invoices'][$now.weekday - 1] }}&limit=10&sort=desc&sort_type=created_utc&after=30d",
        "options": {
          "timeout": 90000
        }
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.5,
      "position": [
        -16,
        -16
      ],
      "id": "d8f5261a-4493-468f-bf4b-e94378a9a324",
      "name": "HTTP Request",
      "retryOnFail": true,
      "maxTries": 4,
      "waitBetweenTries": 15000,
      "alwaysOutputData": false
    },
    {
      "parameters": {
        "fieldToSplitOut": "data",
        "options": {}
      },
      "type": "n8n-nodes-base.splitOut",
      "typeVersion": 1.1,
      "position": [
        176,
        -16
      ],
      "id": "57fff88a-c749-4d69-8b78-f5470290f47c",
      "name": "Split Out"
    },
    {
      "parameters": {
        "jsCode": "const PROMO_FLAIR = /github included|promotion|self-promo|hiring|for hire|showcase/i;\nconst REMOVED_TAG = /\\[\\s*(removed|deleted)/i;\nconst PROMO_TEXT  = /i (built|made|wrote|created|launched) .{0,40}(tool|app|saas|platform|service|extension)|check (out )?my|dm me|link in (bio|comments)|free (template|workflow) (for you|inside)/i;\n\nconst out = [];\nfor (const item of items) {\n  const p = item.json || {};\n  const flair = String(p.link_flair_text || '');\n  const title = String(p.title || '');\n  const body  = String(p.selftext || '');\n  const nComments = Number(p.num_comments || 0);\n\n  // 1. 删帖 / 被版主移除\n  if (p.removed_by_category || p.removal_reason) continue;\n  if (REMOVED_TAG.test(title) || REMOVED_TAG.test(body)) continue;\n\n  // 2. 推广版块\n  if (PROMO_FLAIR.test(flair)) continue;\n\n  // 3. 零评论 = 没人共鸣，需求未经验证\n  if (nComments < 1) continue;\n\n  // 4. 正文太短 = 没有具体信息\n  if (body.trim().length < 80) continue;\n\n  // 5. 疑似软广\n  if (PROMO_TEXT.test(body.slice(0, 700))) continue;\n\n  out.push({ json: {\n    title: p.title,\n    url: p.url,\n    subreddit: p.subreddit,\n    score: Number(p.score || 0),\n    num_comments: nComments,\n    flair: flair,\n    body: body.slice(0, 1100),\n  }});\n}\n\n// 评论越多 = 越多人有同一个痛点\nout.sort((a, b) => b.json.num_comments - a.json.num_comments);\nreturn out;"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        368,
        -16
      ],
      "id": "1d90f62e-f254-4f0d-ba99-e52280d30b09",
      "name": "Pre-filter"
    },
    {
      "parameters": {
        "maxItems": 8
      },
      "type": "n8n-nodes-base.limit",
      "typeVersion": 1,
      "position": [
        560,
        -16
      ],
      "id": "166823b6-6c05-4d8b-b1a1-05fc67622304",
      "name": "Limit"
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Subreddit: r/{{ $json.subreddit }}\nFlair: {{ $json.flair }}\nCommunity response: {{ $json.score }} upvotes, {{ $json.num_comments }} comments\n\nTitle: {{ $json.title }}\n\nBody:\n{{ $json.body }}\n\nExtract the manual, repetitive work this person is complaining about.\nReturn the json object only.",
        "messages": {
          "messageValues": [
            {
              "message": "You are a product-opportunity analyst. Your job is DEMAND DISCOVERY, not lead scoring.\n\nRead one forum post from a small-business community. Find the REPETITIVE, MANUAL work the author complains about.\nThe reader is a freelance automation engineer who builds solutions with n8n and AI.\n\nWhat counts as real demand:\n- A specific recurring task done by hand (copy-paste, spreadsheets, chasing people, re-typing from screenshots or PDFs)\n- A workflow that breaks and costs them time or money\n- \"We do this every week and it takes hours\"\n- They already pay for a workaround (a VA, software, someone part-time)\n\nWhat does NOT count:\n- Vague venting with no concrete task\n- Posts only asking for free advice with no problem described\n- Anyone selling their own product\n- Pure legal, tax or hiring questions with no process to automate\n\ndemand_score 1-10:\n9-10 specific recurring task, already costing money, echoed by commenters\n7-8  specific task, clear time waste, no workaround yet\n5-6  real irritation but the task is vague\n3-4  mostly venting, weak signal\n1-2  no automatable task\n\nReturn ONE json object only. No prose. No markdown fences.\nKeys:\n{\n  \"pain_theme\": \"3-5 word lowercase label of the task, e.g. 'invoice chasing', 'order data entry'\",\n  \"pain_quote\": \"exact phrase from the post showing the manual work, max 25 words\",\n  \"who\": \"small_business_owner | agency | freelancer | ecommerce | other\",\n  \"current_workaround\": \"what they do now, max 12 words, or 'none'\",\n  \"product_idea\": \"one concrete thing an n8n engineer could build, max 18 words\",\n  \"manual_hours\": \"rough hours per week if stated, else 'unknown'\",\n  \"demand_score\": integer 1-10,\n  \"buildable\": integer 1-5 (how easily n8n solves it, 5 = easy)\n}\n\nBe ruthless. If there is no concrete manual task, set demand_score to 1 and pain_theme to \"none\"."
            }
          ]
        },
        "batching": {
          "batchSize": 4
        }
      },
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.9,
      "position": [
        752,
        -16
      ],
      "id": "95d48d27-d086-4773-97f5-6157d32de2dd",
      "name": "Basic LLM Chain",
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "model": {
          "__rl": true,
          "value": "",
          "mode": "list",
          "cachedResultName": "",
          "cachedResultUrl": ""
        },
        "responsesApiEnabled": false,
        "options": {
          "maxTokens": 600,
          "responseFormat": "json_object",
          "temperature": 0.2,
          "timeout": 45000,
          "maxRetries": 1
        }
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1.3,
      "position": [
        752,
        208
      ],
      "id": "83fd4274-0161-4800-adab-224814cc9cde",
      "name": "OpenAI Chat Model"
    },
    {
      "parameters": {
        "jsCode": "const THEME_MIN_SCORE = 3;\n\nfunction esc(s) {\n  return String(s == null ? '' : s)\n    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')\n    .replace(/\"/g, '&quot;');\n}\n\nconst buckets = new Map();\nlet analyzed = 0, skipped = 0;\n\nfor (const item of items) {\n  const j = item.json || {};\n  if (j.error) { skipped++; continue; }\n  const theme = String(j.pain_theme || '').trim();\n  const score = Number(j.demand_score == null ? 1 : j.demand_score);\n  if (!theme || theme.toLowerCase() === 'none' || score < THEME_MIN_SCORE) { skipped++; continue; }\n  analyzed++;\n\n  const key = theme.toLowerCase().replace(/[^a-z0-9 ]/g, '').replace(/\\s+/g, ' ').trim();\n  if (!buckets.has(key)) {\n    buckets.set(key, {\n      theme, posts: 0, scoreMax: 0, scoreSum: 0, comments: 0,\n      buildable: 0, quotes: [], ideas: [], who: new Set(), urls: [], workaround: '',\n    });\n  }\n  const b = buckets.get(key);\n  b.posts++;\n  b.scoreMax = Math.max(b.scoreMax, score);\n  b.scoreSum += score;\n  b.comments += Number(j.num_comments || 0);\n  b.buildable = Math.max(b.buildable, Number(j.buildable || 0));\n  if (j.pain_quote) b.quotes.push(String(j.pain_quote).slice(0, 160));\n  if (j.product_idea && !b.ideas.includes(String(j.product_idea))) {\n    b.ideas.push(String(j.product_idea).slice(0, 170));\n  }\n  if (j.who) b.who.add(String(j.who));\n  if (j.url && !b.urls.includes(String(j.url))) b.urls.push(String(j.url));\n  if (!b.workaround && j.current_workaround && j.current_workaround !== 'none') {\n    b.workaround = String(j.current_workaround).slice(0, 90);\n  }\n}\n\nconst themes = [...buckets.values()].map(b => ({\n  theme: b.theme,\n  posts: b.posts,\n  score: b.scoreMax,\n  avg: Math.round((b.scoreSum / b.posts) * 10) / 10,\n  comments: b.comments,\n  buildable: Math.min(5, b.buildable),\n  quote: b.quotes.sort((x, y) => y.length - x.length)[0] || '',\n  idea: b.ideas[0] || '',\n  workaround: b.workaround,\n  who: [...b.who].slice(0, 2).join(' / '),\n  url: b.urls[0] || '',\n})).sort((a, b) => (b.comments + b.score * 4) - (a.comments + a.score * 4)).slice(0, 6);\n\n// ---- 日期 ----\nconst d = $now.setZone('Asia/Shanghai');\nconst ymd = d.toFormat('yyyy-MM-dd');\nconst md  = d.toFormat('MM-dd');\nconst dow = ['周一','周二','周三','周四','周五','周六','周日'][d.weekday - 1];\n\n// ---- HTML ----\nconst F = \"font-family:-apple-system,Segoe UI,Arial,sans-serif\";\nlet html = '';\nhtml += '<div style=\"' + F + ';font-size:14px;color:#222;max-width:900px\">';\nhtml += '<h2 style=\"margin:0 0 4px\">需求雷达 · ' + ymd + ' · ' + dow + '</h2>';\nhtml += '<p style=\"color:#666;font-size:12px;margin:0 0 16px\">'\n      + '抓取 ' + items.length + ' 条 → 有效 ' + analyzed + ' 条 → 聚类成 '\n      + themes.length + ' 个需求主题（跳过 ' + skipped + ' 条）</p>';\n\nif (themes.length === 0) {\n  html += '<div style=\"padding:14px;background:#fff4e5;border-left:4px solid #f90;margin:12px 0\">'\n        + '<b>本轮没有挖到明确的可自动化需求。</b><br/>'\n        + '<span style=\"font-size:13px;color:#555\">这不代表流程出错。'\n        + '可能是本周这个查询没有真人在抱怨，或抱怨里没有可自动化的具体任务。</span></div>';\n} else {\n  html += '<table cellpadding=\"9\" cellspacing=\"0\" style=\"border-collapse:collapse;width:100%;font-size:13px;border:1px solid #ddd\">';\n  html += '<tr style=\"background:#1a1a1a;color:#fff;text-align:left\">'\n        + '<th>需求主题</th><th>帖</th><th>评论</th><th>强度</th><th>好做</th>'\n        + '<th>他们现在怎么做</th><th>你能做什么</th></tr>';\n  for (const t of themes) {\n    html += '<tr style=\"border-top:1px solid #eee\">'\n          + '<td><b>' + esc(t.theme) + '</b><br/>'\n          + '<small style=\"color:#999\">' + esc(t.who) + '</small></td>'\n          + '<td align=\"center\">' + t.posts + '</td>'\n          + '<td align=\"center\">' + t.comments + '</td>'\n          + '<td align=\"center\"><b style=\"color:' + (t.score >= 7 ? '#0a0' : '#c80') + '\">'\n          + t.score + '</b></td>'\n          + '<td align=\"center\">' + '\\u2605'.repeat(t.buildable) + '</td>'\n          + '<td style=\"color:#555\">' + esc(t.workaround || '—') + '</td>'\n          + '<td><b>' + esc(t.idea) + '</b></td>'\n          + '</tr>';\n  }\n  html += '</table>';\n\n  html += '<h3 style=\"margin:22px 0 8px;font-size:15px\">原话证据</h3>';\n  for (const t of themes) {\n    if (!t.quote) continue;\n    html += '<div style=\"margin:0 0 10px;padding:9px 12px;background:#f6f8fa;'\n          + 'border-left:3px solid #0a0\">'\n          + '<b style=\"font-size:13px\">' + esc(t.theme) + '</b>'\n          + '<div style=\"font-size:13px;color:#333;margin:3px 0;font-style:italic\">'\n          + '“' + esc(t.quote) + '”</div>'\n          + (t.url ? '<a href=\"' + esc(t.url) + '\" style=\"font-size:12px;color:#06c\">原帖 →</a>' : '')\n          + '</div>';\n  }\n}\nhtml += '<p style=\"color:#bbb;font-size:11px;margin-top:20px\">'\n      + 'n8n 需求雷达 · 每日轮换版块与查询词 · 数据源 arctic-shift</p></div>';\n\nconst subject = themes.length === 0\n  ? '需求雷达 ' + md + ' · 本轮无需求'\n  : '需求雷达 ' + md + ' · ' + themes.length + ' 个主题（最强：' + themes[0].theme + '）';\n\nreturn [{\n  json: {\n    themes, theme_count: themes.length,\n    posts_analyzed: analyzed, posts_skipped: skipped,\n    posts_fetched: items.length,\n    subject, html,\n  },\n}];"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1024,
        -16
      ],
      "id": "f57d82a2-dcc2-4646-a075-5b2f947c7ee5",
      "name": "Build Report"
    },
    {
      "parameters": {
        "fromEmail": "2130353792@qq.com",
        "toEmail": "2130353792@qq.com",
        "subject": "={{ $json.subject }}",
        "html": "={{ $json.html }}",
        "options": {
          "appendAttribution": false
        }
      },
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [
        1168,
        -16
      ],
      "id": "8cc987ca-8bff-42a7-bc07-f06611b67b67",
      "name": "Send an Email",
      "webhookId": "38c0d6cd-02ab-4e2f-9e00-708e47d3b982",
      "credentials": {
        "smtp": {
          "id": "Nj8iLuzixOvrMIa6",
          "name": "SMTP account"
        }
      }
    }
  ],
  "pinData": {},
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "HTTP Request",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "HTTP Request": {
      "main": [
        [
          {
            "node": "Split Out",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Split Out": {
      "main": [
        [
          {
            "node": "Pre-filter",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Pre-filter": {
      "main": [
        [
          {
            "node": "Limit",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Limit": {
      "main": [
        [
          {
            "node": "Basic LLM Chain",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Basic LLM Chain": {
      "main": [
        [
          {
            "node": "Build Report",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "OpenAI Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "Basic LLM Chain",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Build Report": {
      "main": [
        [
          {
            "node": "Send an Email",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "binaryMode": "separate",
    "timeSavedMode": "fixed",
    "timezone": "Asia/Shanghai",
    "callerPolicy": "workflowsFromSameOwner",
    "availableInMCP": false
  },
  "versionId": "c286f383-01d2-4211-b1c6-0ded3cd29563",
  "meta": {
    "instanceId": "67ae6109d4aee16c8174f2d986045cb8a99efb0e0a8e71293d1a9682b9bbb2d3"
  },
  "nodeGroups": [],
  "id": "sDILyN0bllThu5CP",
  "tags": []
}
