---
title: 使用Cloudlare配置openai转发并部署网页版ChatGPT
date: 2023-04-14 16:54:37
tags: CloudFlare
---

# 使用CloudFlare配置openai转发并部署网页版ChatGPT

众所周知, 在国内访问 ChatGPT 有各种各样的麻烦, 各种代理的 IP 被封也就算了, 乱登录可能账号都难保, 尤其是现在毛子的电话号码网站没了的情况. 实在是无法忍受代理粗糙的性能, 我决定在自己的电脑上搭建一个网页版的 ChatGPT, 通过调用 API 实现大部分的功能, 且全程合法合规, 甚至不会用到 VPS.

大致的思路分为两步: 首先openai 的 API 是无法直接访问的, 其次拿到了 API 相当于是最底层的接口, 直接用 `POST` 方法人工操作显然很愚蠢且麻烦, 需要套一层壳. 

注意, 这两步都一定要亲自实现, 否则会有极大的安全问题. 因为所有的 `POST` 方法都会把内容明文传输. 我记得前段时间明文传输密码的 `P大树洞` 还出来洗地说 HTTPS 保障了安全, 简直是无稽之谈. `HTTPS` 只能保证端到端的安全, 甚至连访问的 IP 都清晰可见, 两端肯定是要解密的呀. 要是用了别人的 API 代理, 对方可以直接看到你的 API Key 以及所有的问答内容; 要是用了别人的前端这些信息依然可见. 如果别人不怀好意使用你的 Key 耗费了大量的 Token, 根本找不到人哭.

## 利用 Cloudflare 中转 API

虽然 openai 的 API 是无法直接访问, 这个调用比直接上网页版的限制还是宽松得多. 解铃还须系铃人, 既然 openai 选择了使用 Cloudflare 来做网页防护和 IP 鉴定, 我们干脆直接让 Cloudflare 帮我们做这个代理.

[chatgptProxyAPI](https://github.com/x-dr/chatgptProxyAPI) 帮助我们实现了这个功能. 其核心思想是在 CloudFlare 中设置一个无服务器的服务, 执行一些简单的功能.

创建一个worker, 代码部分填充如下:

```
const TELEGRAPH_URL = 'https://api.openai.com';

addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url);
  url.host = TELEGRAPH_URL.replace(/^https?:\/\//, '');

  const modifiedRequest = new Request(url.toString(), {
    headers: request.headers,
    method: request.method,
    body: request.body,
    redirect: 'follow'
  });

  const response = await fetch(modifiedRequest);
  const modifiedResponse = new Response(response.body, response);

  // 添加允许跨域访问的响应头
  modifiedResponse.headers.set('Access-Control-Allow-Origin', '*');

  return modifiedResponse;
}
```

右边会显示 404, 但是没关系这是正常现象.

部署之后, 在 `Trigger` 中设置自己想要的 URL, 方便我们调用.

直接访问这个链接的访问如下:

```
{
  "error": {
    "message": "Invalid URL (GET /)",
    "type": "invalid_request_error",
    "param": null,
    "code": null
  }
}
```

这是因为我们调用时并没有传入合法的参数, 如果真的需要, 可以使用以下 Python 代码测试代理是否工作:

``` PYTHON
url = "https://[your trigger URL]/v1/chat/completions"
api_key = 'sk-xxxxxxxxxxxxxxxxxxxx'

model_engine = "gpt-3.5-turbo" 
prompt = "介绍一下你自己" 
role = "user"
response = requests.post( "https://[your trigger URL]/v1/chat/completions", \
     headers={ "Authorization": "Bearer {}".format(api_key), "Content-Type": "application/json", }, \
    json={"model": "gpt-3.5-turbo", "messages": [{"role": role, "content": prompt}]}, ) 
try:
    print(response.json()['choices'][0]['message']['content'])
except:
    print(response.json())
```

有一点值得注意, 就是在官方文档中你可以看到两类模型, 一类是 `GPT3.5`, 另一类是 `Davinci`, 两类的调用参数和 URL 都不一样. 一般来说建议使用 `gpt-3.5-turbo`, 因为这个模型能节省大多的 Token.

这里再给一个达芬奇模型的例子:

``` PYTHON
import requests 
api_key ='sk-xxxxxxxxxxxxxxxxxxxx' 
model_engine = "text-davinci-003" 
prompt = "What is DataFrame concat function？" 
response = requests.post( "https://[your trigger URL]/v1/completions", \
     headers={ "Authorization": "Bearer {}".format(api_key), "Content-Type": "application/json", }, \
    json={ "model": model_engine, "prompt": prompt, "max_tokens": 256, "temperature": 0.5, }, ) 
try:
    print(response.json()["choices"][0]['text'])
except:
    print(response.json())
```

## 搭建 web 版本的 ChatGPT

推荐使用这个 Github 上 Star 最高的[框架](https://github.com/Chanzhaoyu/chatgpt-web), 它实现了基本上全部网页原版的内容.

我在使用时直接用封装好的文件会有小问题, 所以使用了 npm 构建. 这样会产生两个端口, 一个是用于后端服务的, 一个是提供前端网页的. 我们在做 nginx 转发的时候转发的应当是前端的端口. 如果不能逐字显示文本, 考虑关闭 nginx 的缓存就好.




















