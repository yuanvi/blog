---
published: true
title: python快速实现http接口
layout: post
tags: [python, http]
categories: [Python, Web]
---
平时常用```python -m SimpleHTTPServer 8080```来创建一个http服务共享文件，但有时候会有数据需要做个接口来给别人调用，或者是去查某个服务需要有一个http回调接口，很多时候这些接口还不是长期的，去装一个框架或者http服务感觉又划不来。看了一下，其实SimpleHTTPServer模块就可以满足需求 [参考文档](https://docs.python.org/2/library/simplehttpserver.html#SimpleHTTPServer.SimpleHTTPRequestHandler)，简单实现如下：

```python
def interface():
    socket.setdefaulttimeout(10)
    import BaseHTTPServer
    class HttpHandler(BaseHTTPServer.BaseHTTPRequestHandler):

        def do_POST(self):
            try:
                len = self.headers.get('Content-Length', 0)
                content = self.rfile.read(int(len))

                #do something here

                resp = '''HTTP/1.1 200 OK\r\nContent-Length: 0\r\n'''
                self.wfile.write(resp)
            except Exception as exp:
                resp = '''HTTP/1.1 500 Internal Server Error\r\nContent-Length: 0\r\n'''
                self.wfile.write(resp)

    httpd = BaseHTTPServer.HTTPServer(CALLBACK_SERVER_ADDR, HttpHandler)
    # if need
    #for i in range(THREAD_NUM):
    #    threading.Thread(target = httpd.serve_forever).start()
    httpd.serve_forever()
```

必须要设置read socket的超时时间,否则很容易所有线程就都卡死了。
