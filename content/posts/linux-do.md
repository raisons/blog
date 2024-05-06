---
title: '一个Linux.do自动点赞、阅读帖子的脚本'
date: 2024-05-06T12:22:00+08:00
tags: ["linux.do","python","discourse","auto"]
---



**仅限技术研究，请勿刷赞刷阅读**



> 核心代码参考了 https://114114.xyz/article/discourse_org_read_and_liked



使用协程，但是并未使用并发请求，仍是同步执行，方便后期添加异步任务。



配置文件`conf.json`

```json
{
  "csrf-token": "",
  "cookies": {
    "_forum_session": "",
    "cf_clearance": "",
    "_t": ""
  },
  "connect-cookies": {
    "auth.session-token": "",
    "cf_clearance": ""
  }
}
```

代码

```python
#!/usr/bin/env python
import httpx
import asyncio
import json
import copy
from pathlib import Path
from pyquery import PyQuery

HEADERS = {
    "Accept": "application/json, text/javascript, */*; q=0.01",
    "Host": None,
    "Discourse-Logged-In": "true",
    "Discourse-Present": "true",
    "Discourse-Track-View": "true",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                  "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.4.1 "
                  "Safari/605.1.15",
    "Referer": None,
    "Sec-Fetch-Dest": "empty",
    "Sec-Fetch-Mode": "cors",
    "Sec-Fetch-Site": "same-origin",
    "X-Requested-With": "XMLHttpRequest",
}

CONNECT_HEADERS = {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Host": "connect.linux.do",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "none",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                  "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.4.1 "
                  "Safari/605.1.15"
}


class Topic:

    def __init__(self, client):
        self.client = client

    async def _fetch(self, url, **params):
        response = await self.client.get(url, params=params)
        return response.json()["topic_list"]["topics"]

    async def latest(self, order="created"):
        return await self._fetch("/latest.json", order=order)

    async def top(self, period="daily"):
        return await self._fetch("/top.json", period=period)

    async def new(self):
        return await self._fetch("/new.json")

    async def unread(self):
        return await self._fetch("/unread.json")

    async def timings(self, data):
        return await self.client.post("/topics/timings", data=data)

    async def detail(self, topic_id: int):
        response = await self.client.get(
            f"/t/{topic_id}.json?track_visit=true"
        )
        return response.raise_for_status().json()


class Discourse:

    def __init__(self, domain: str):
        self.domain = domain
        self.client = self.get_client()
        self.topic = Topic(self.client)

    @property
    def conf(self) -> dict:
        if not hasattr(self, "_conf"):
            file = Path(__file__).parent / "conf.json"
            if not file.exists() or not file.is_file():
                raise FileNotFoundError("conf.json not exist")
            with file.open("r") as fp:

                conf = json.load(fp)
                setattr(self, "_conf", conf)

        return getattr(self, "_conf")

    def get_client(self):
        headers = copy.deepcopy(HEADERS)
        headers["Referer"] = f"{self.domain}/"
        headers["Host"] = self.domain
        headers["X-CSRF-Token"] = self.conf["csrf-token"]
        client = httpx.AsyncClient(
            base_url=self.domain,
            cookies=self.conf["cookies"],
            headers=headers
        )
        return client

    def connect(self):
        response = httpx.get(
            "https://connect.linux.do",
            cookies=self.conf["connect-cookies"],
            headers=CONNECT_HEADERS
        )

        html = response.content.decode()

        doc = PyQuery(html)
        info = doc("h1").text()
        info = info.split("，")[1]
        info = info.split(" ")[:-1]
        # print(info)
        print("Nickname:", info[0])
        print("Username:", info[1][1:-1])
        print("Level:", info[2])

    def log(self, *args, **kwargs):
        print(*args, **kwargs)

    async def summary(self, username):
        response = await self.client.get(f"/u/{username}/summary.json")
        user = response.json()["user_summary"]
        print("用户名:", username)
        print("访问天数:", user["days_visited"])
        print("帖子阅读时间:", f"{user['time_read'] // 3600}小时")
        print("浏览的话题数:", user["topics_entered"])
        print("帖子阅读数量:", user["posts_read_count"])
        print("访问天数:", user["days_visited"])
        print("送出点赞:", user["likes_given"])
        print("收到点赞:", user["likes_received"])
        print("创建话题:", user["topic_count"])
        print("创建帖子:", user["post_count"])

    async def topics_timings(
        self,
        topic_id: int,
        highest_post_num: int,
        last_read_post_num: int,
        read_time: int = 10000
    ):
        if last_read_post_num is None:
            last_read_post_num = 0
        read_count = highest_post_num - last_read_post_num
        data = {
            "topic_id": topic_id,
            "topic_time": 0
        }
        for i in range(last_read_post_num, highest_post_num + 1):
            data["topic_time"] += read_time
            data[f'timings[{i}]'] = data["topic_time"]

        response = await self.topic.timings(data=data)
        if response.status_code == 200:
            return read_count
        return 0

    async def like_post(self, post_id):
        response = await self.client.put(
            f"/discourse-reactions/posts/{post_id}/custom-reactions/heart/toggle.json"
        )

        if response.status_code == 200:
            self.log(f"Successfully liked post: {post_id}")
            return 1

        elif response.status_code == 429:
            # self.states['liked_topics'] = 75
            # now = datetime.datetime.now()
            resp = response.json()
            wait_seconds = resp["extras"]["wait_seconds"]
            # target_time = now + datetime.timedelta(seconds=wait_seconds)
            # self.states['liked_wait_until'] = target_time.isoformat()
            self.log(f'当前点赞到上限，再等待{resp["extras"]["time_left"]}')

        else:
            self.log(f"Failed to like post: {post_id}")
            self.log(response.status_code, response.text)

        return 0

    async def auto_read_posts(self, topics):
        total_read_posts = 0
        for topic in topics:
            read_count = await self.topics_timings(
                topic["id"],
                topic["highest_post_number"],
                topic.get("last_read_post_number", None)
            )
            total_read_posts += read_count
            self.log(f"读取topic {topic['id']}, {read_count}条评论")
            await asyncio.sleep(1)

        self.log(f'成功标记 {total_read_posts} posts 为读取')

    async def auto_like(self):
        total_liked_topics = 0
        topics = await self.topic.new()
        for topic in topics:
            if topic["unseen"]:
                topic_info = await self.topic.detail(topic["id"])
                post_stream = topic_info["post_stream"]
                first_post = post_stream["posts"][0]

                like_count = await self.like_post(first_post["id"])
                total_liked_topics += like_count

                await asyncio.sleep(1)
        self.log(f"送出点赞：{total_liked_topics}个")

    async def run(self, username=None):
        # await self.login()
        # await self.auto_read_posts(await self.topic.unread())
        # await self.auto_read_posts(await self.topic.new())
        # await self.auto_read_posts(await self.topic.top())
        # await self.auto_read_posts(await self.topic.latest())
        await self.auto_like()
        if username:
            await self.summary(username)


if __name__ == '__main__':
    d = Discourse("https://linux.do")
    asyncio.run(d.run())

```



