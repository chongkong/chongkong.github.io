---
layout: post
title: "Slack RTM 400 Bad Request"
date: 2016-11-23 21:42:41 +0900
categories: bugfix 
---

slack 봇을 만들던 중 `aiohttp` 라이브러리에서 제공하는 websocket client가 동작하지 않는 상황이 있었다. 문서에서 알려준 대로 잘 사용했는데 Bad Request 응답이 왔다.

``` python
async with self.session.ws_connect(url) as ws:
    async for msg in ws:
        if msg.type == aiohttp.WSMsgType.TEXT:
            print('msg', msg.data)
```

그런데 분명 한 [미디엄 글](https://medium.com/@greut/a-slack-bot-with-pythons-3-5-asyncio-ad766d8b5d8f#.4ybojq2ja)에서는 위와 같은 방법으로 websocket을 연결하는 코드를 사용하고 있었다.
예전 버전에서는 되었던 것이라 생각하여 버전을 이리 저리 바꿔보다 보니 1.0.5 버전에서는 잘 동작하지만 1.1.0 버전부터는 동작하지 않는다. 좀 더 구체적으로 [e81a4781d8](https://github.com/KeepSafe/aiohttp/commit/e81a4781d80b614da572c2c18635831ad024126b) 
커밋부터 동작하지 않는 것을 확인했다. 해당 커밋의 변경사항은 URL을 직접 파싱하지 않고 [YARL](https://github.com/aio-libs/yarl)이라는 라이브러리를 사용하는 내용이다.

실제로 디버거를 키고 Request를 찍어 보니 URL이 다르게 찍히고 있었다.

```
# v1.0.5
wss://mpmulti-xu7e.slack-msgs.com/websocket/mWJePSkY9j_1Q4iiFSqdt...c0ZWCVZQY_DbY=

# v1.1.0
wss://mpmulti-xu7e.slack-msgs.com/websocket/mWJePSkY9j_1Q4iiFSqdt...c0ZWCVZQY_DbY%3D
```

즉 YARL 라이브러리를 사용하면서 `=`이 `%3D`로 urlencode 되도록 바뀌었다. 문제는 `rtm.start` 슬랙 RTM API에서 반환하는 URL은 항상 `=`가 마지막에 붙어 있고 (base64인코딩) 
슬랙 API에서는 `=` 대신 `%3D`를 사용하면 Bad Request를 보낸다는 것이다!

일단 Slack 에 문의를 넣어두었고 당분간 1.0.5 버전을 사용하는걸로 마무리를 지었다.
