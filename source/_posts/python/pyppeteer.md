---
date: 2018-06-21
title: Pypppeteer
categories: Python
tag: pypppeteer 
---

之前做爬虫或者浏览器自动化操作都用selenium ，再配合docker可以进行分布式部署,但是selenium太耗性能,这里有另外个选择[puppeteer](https://github.com/GoogleChrome/puppeteer)谷歌浏览器在17年自行开发了Chrome Headless特性，并与之同时推出了puppeteer，可以理解成我们日常使用的Chrome的无界面版本以及对其进行操控的js接口套装



### pyppeteer

``` php
import asyncio
from pyppeteer import launch

async def main():
    browser = await launch({'headless':False})
    page = await browser.newPage()
    await page.goto('http://http://nifty.dwyjr.cn')
    await page.type('input[name="email"]', 'yujiarong@sailvan.com', {"delay": 10});
    await page.type('input[name="password"]', 'yujiarong', {"delay": 10});
    await page.click('#container > div.cls-content > div > div.panel-body > form > button')
    await page.screenshot({'path': 'example.png'})
    cookies =  await page.cookies() 
    await browser.close()
    print( cookies)

loop = asyncio.get_event_loop()
tasks = [
	asyncio.ensure_future(main()),
	asyncio.ensure_future(main()),
	asyncio.ensure_future(main())
]


loop.run_until_complete(asyncio.wait(tasks))
```
pyppeteer支持异步，具体操作可以直接看[puppeteer](https://github.com/GoogleChrome/puppeteer)的文档 ，pyppeteer的命令差不多

* 用来截屏 
* 登陆获取cookie
* 爬去异步渲染的页面信息
* 并发操作默写不可描述的东西 嘿嘿