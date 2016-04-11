---
published: true
title: flask中static_folder实现
layout: post
tags: [python, flask]
categories: [python, web]
---
一个比较奇葩的需求，希望在flask中指定了static_folder之后，对static folder中目录的请求能实现类似default page的效果，看了一下实现，发现flask并不支持。

在初始化Flask对象时，可以指定static_folder和static_url_path，代码大致如下：

```python
class Flask(_PackageBoundObject):
    def __init__(self, import_name, static_path=None, static_url_path=None,         
            static_folder='static', template_folder='templates',                    
            instance_path=None, instance_relative_config=False):                    
        ...
        if self.has_static_folder:                                                  
            self.add_url_rule(self.static_url_path + '/<path:filename>',            
                    endpoint='static',                                              
                    view_func=self.send_static_file)
```

可以看到，实际上flask是将send_static_file函数通过add_url_rule注册给了static这个endpoint ("endpoint" is an identifier that is used in determining what logical unit of your code should handle the request.) 。send_static_file这个函数在helpers.py中，它简单的调用了send_from_directory尝试发送我们设置的static folder中的文件。

```python
def send_static_file(self, filename):
    cache_timeout = self.get_send_file_max_age(filename)
    return send_from_directory(self.static_folder, filename,
                                  cache_timeout=cache_timeout)

def send_from_directory(directory, filename, **options):
    filename = safe_join(directory, filename)
    if not os.path.isfile(filename):
        raise NotFound()
    options.setdefault('conditional', True)
    return send_file(filename, **options)
```

看到这里已经可以确定flask不支持这种设置了，那自己简单实现如下：

```python
@app.route('/', defaults = {'path': ''})
@app.route('/<path:path>')
def router(path):
    path = safe_join(STATIC_DIR, path)
    if os.path.isdir(path):
        path = safe_join(path, 'index.html')    #default page is index.html
    if os.path.isfile(path):
        return send_file(path)
    return "page not found", 404 
```