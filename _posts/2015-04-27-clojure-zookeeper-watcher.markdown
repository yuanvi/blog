---
published: true
title: clojure实现zookeeper watcher小例子
layout: post
tags: [clojure, zookeeper]
categories: [Clojure, 分布式计算]
---
用clojure基于curator实现一个zookeeper node watcher的小例子：

```clojure
(ns clolein.core
    (:import [org.apache.curator.framework CuratorFramework CuratorFrameworkFactory]
             [org.apache.curator.framework.api CuratorListener]
             [org.apache.curator RetryPolicy]
             [org.apache.zookeeper Watcher WatchedEvent])
    (:gen-class))

(def watcher
    (reify Watcher
        (process [this event]
            (println "listener")
            (println (.getPath event)))))

(def curator 
    (.build
        (.retryPolicy
            (.connectString 
                (CuratorFrameworkFactory/builder)
                "127.0.0.1:2181")
            (reify RetryPolicy 
                (allowRetry [this count elapsed sleeper] false)))))

(defn to-bytes [str]
    (byte-array (map byte str)))

(defn watch-zookeeper []
    (.start curator)
    (.forPath 
        (.create curator)
        "/test" 
        (to-bytes "from clojure"))
    (.forPath
        (.usingWatcher 
            (.getChildren curator)
            watcher)
        "/test"))

(defn -main []
    (watch-zookeeper)
    (loop []
        (println "in loop")
        (Thread/sleep 1000)
        (recur))
    (println "main finish"))
```
