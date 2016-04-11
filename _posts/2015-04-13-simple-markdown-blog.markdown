---
published: true
title: 快速实现markdown博客(2)
layout: post
tags: [python, flask]
categories: [python, web]
---
基于flask快速实现简单粗暴markdown的博客，render主要在前端来做，用 [SimpleMDE](https://github.com/NextStepWebs/simplemde-markdown-editor)，那么后端仅用维护文件目录和render出html文件就行。总共仅需两个文件：

```python
#!/usr/bin/env python
#coding=utf8

import os
from flask import *

import render
render.render_all()

app = Flask(__name__)
app.debug = True

AUTHKEY = "viblog"
BASE_DIR = os.path.dirname(os.path.realpath(__file__))
ROOT_DIR = os.path.join(BASE_DIR, 'root')

@app.route('/_editor', methods = ['POST'])
def editor():
    if request.form.get('auth') != AUTHKEY:
        return "Don't you dare!"

    if render.create_archive(request.form.get('markdown')):
        render.render_all()
        return "Success!"
    else:
        return "Something wrong!"
        
@app.route('/', defaults = {'path': ''})
@app.route('/<path:path>')
def router(path):
    path = safe_join(ROOT_DIR, path)
    if os.path.isdir(path):
        path = safe_join(path, 'index.html')
    if os.path.isfile(path):
        return send_file(path)
    return "page not found", 404

app.run("0.0.0.0", 8080)
```


```python
#!/usr/bin/env python
#coding=utf8

import os
import sys
import time
import json

from jinja2 import Environment, FileSystemLoader

ROOT_DIR = os.path.join(os.getcwd(), "root")
ARCHIVE_PATH = os.path.join(os.getcwd(), "root/archives")
TEMPLATE_PATH = os.path.join(os.getcwd(), "templates")

_env = Environment(
    autoescape = False,
    loader = FileSystemLoader(TEMPLATE_PATH),
    trim_blocks = False)

def render_template(template, **kwargs):
    return _env.get_template(template).render(kwargs)

def parse_archive_property(property_line):
    arr = property_line.decode("utf8").split('->')
    key = arr[0][1:].strip().lower()
    if key == "title":
        value = arr[1].strip()
    else:
        value = json.loads(arr[1].strip(), encoding = "utf8")
    return key, value

def parse_archive(markdown):
    content = markdown.strip().split('\n')
    archive = {}
    for i in range(len(content)):
        if content[0].startswith(':'):
            k, v = parse_archive_property(content[0])
            archive[k] = v 
        elif content[0].strip():
            break
        content.pop(0)

    archive['content'] = ''.join(content).decode("utf8")
    return archive

def render_archives():
    archives = []
    for dirname in os.listdir(os.path.join(ROOT_DIR, 'archives')):
        fn = os.path.join(ROOT_DIR, 'archives', dirname, 'archive.md')
        if os.path.exists(fn):
            archive = parse_archive(open(fn).read())
            archive['link'] = "archives/" + dirname
            archives.append(archive)
            content = render_template('archive.html', archive = archive).encode('utf8')
            filename = os.path.join(ROOT_DIR, 'archives', dirname, 'index.html')
            open(filename, 'w').write(content)
        else:
            print >>sys.stderr, 'render ' + dirname + ' error!'
    return archives

def create_archive(markdown):
    archive = parse_archive(markdown)
    if not archive:
        return False
    timestamp = str(int(time.time()))
    path = os.path.join(ROOT_DIR, 'archives', timestamp)
    os.mkdir(path)
    open(path + '/archive.md', 'w').write(markdown.encode('utf8'))

def render_all():
    archives = render_archives()

    content = render_template("index.html", archives = archives).encode("utf8")
    open(os.path.join(ROOT_DIR, "index.html"), "w").write(content)
            
if __name__ == "__main__":
    render_all()
```
