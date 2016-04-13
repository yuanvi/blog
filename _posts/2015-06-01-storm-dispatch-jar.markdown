---
published: true
title: storm源码分析 - Jar文件分发
layout: post
tags: [storm, 源码分析, java, clojure, thrift]
categories: [Java, Clojure, 分布式计算]
---

在使用storm过程中，通常用类似下面的代码来提交topology到集群，首先，storm需要将jar包分发到分配的supervisor机器上，这篇文章主要分析storm中Jar文件分发的过程。

```java
TopologyBuilder builder = new TopologyBuilder();

builder.setSpout("1", new TestWordSpout(true), 5);
builder.setSpout("2", new TestWordSpout(true), 3);
builder.setBolt("3", new TestWordCounter(), 3)
         .fieldsGrouping("1", new Fields("word"))
         .fieldsGrouping("2", new Fields("word"));
builder.setBolt("4", new TestGlobalCount())
         .globalGrouping("1");

Map conf = new HashMap();
conf.put(Config.TOPOLOGY_WORKERS, 4);

StormSubmitter.submitTopology("mytopology", conf, builder.createTopology());
```

查看StormSubmitter的代码，最终submitTopology会调用submitJarAs函数通过nimbus的thrift接口上传本地jar包，这个过程中调用了三个nimbus thrift接口，首先通过beginFileUpload获取文件在nimbus上的写入路径，这个路径在之后提交topology时要用到；然后，调用uploadChunk函数将数据分段上传，默认分段大小为300K；最后，通过finishFileUpload通知nimbus文件上传完毕。代码流程如下:

```java
// 获取thrift接口client
NimbusClient client = NimbusClient.getConfiguredClientAs(conf, asUser);

// thrift接口
String uploadLocation = client.getClient().beginFileUpload();
LOG.info("Uploading topology jar " + localJar + " to assigned location: " + uploadLocation);

// THRIFT_CHUNK_SIZE_BYTES = 307200, is.read()每次读取300K的数据
BufferFileInputStream is = new BufferFileInputStream(localJar, THRIFT_CHUNK_SIZE_BYTES);

// 每次传输300K的数据
while(true) {
    byte[] toSubmit = is.read();
    if(toSubmit.length==0) break;
    client.getClient().uploadChunk(uploadLocation, ByteBuffer.wrap(toSubmit));
}
client.getClient().finishFileUpload(uploadLocation);
```

这里有个传递localJar路径的细节：提交topology时，使用```storm jar target/top-0.1.0.jar TopoloyMain```命令，实际上storm命令使用bin/storm.py文件调用java来运行jar包，"jar"子命令在storm.py中对应的jar函数如下，它把运行的jar包路径写入命令行的jvmopts中，那么在java代码中只需要调用System.getProperty("storm.jar")就可以获取到文件路径。

```python
def jar(jarfile, klass, *args):
    """Syntax: [storm jar topology-jar-path class ...]"""
    exec_storm_class(
        klass,
        jvmtype="-client",
        extrajars=[jarfile, USER_CONF_DIR, STORM_BIN_DIR],
        args=args,
        daemon=False,
        jvmopts=JAR_JVM_OPTS + ["-Dstorm.jar=" + jarfile])
```

接下来，分析一下numbus端thrift server中对应接口的实现，在源码中storm.thrift文件里，nimbus相关接口定义如下：

```thrift
service Nimbus {
  string beginFileUpload() throws (1: AuthorizationException aze);
  void uploadChunk(1: string location, 2: binary chunk) throws (1: AuthorizationException aze);
  void finishFileUpload(1: string location) throws (1: AuthorizationException aze);

  //@deprecated beginBlobDownload does that
  string beginFileDownload(1: string file) throws (1: AuthorizationException aze);
  BeginDownloadResult beginBlobDownload(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  //can stop downloading chunks when receive 0-length byte array back
  binary downloadChunk(1: string id) throws (1: AuthorizationException aze);
}
```

storm项目源码在github上统计有超过77%为Java，但其中绝大部分是自动生成的，核心的实现语言其实是clojure，nimbus的thrift接口Nimbus$Iface也是在daemon/nimbus.clj文件中实现的，这里整理一下相关接口实现代码：

```clojure
(reify Nimbus$Iface

  ;; 返回一个保存的文件路径
  (beginFileUpload [this]
    ;; Java metrics mark
    (mark! nimbus:num-beginFileUpload-calls) 
    ;; 检查fileUpload操作权限
    (check-authorization! nimbus nil nil "fileUpload")
    ;; 调用inbox函数根据配置和uuid生成下面示例文件路径，并且创建文件夹
    ;; Example: [Config.STORM_LOCAL_DIR]/nimbus/inbox/stormjar-[uuid].jar
    (let [fileloc (str (inbox nimbus) "/stormjar-" (Utils/uuid) ".jar")]
      ;; nimbus uploaders是一个TimeCacheMap，存储的key超时会自动删除，超时时间由NIMBUS-FILE-COPY-EXPIRATION-SECS设定
      ;; 这里用文件名做key存储了对应的FileOutputStream对象
      (.put (:uploaders nimbus)
            fileloc
            (Channels/newChannel (FileOutputStream. fileloc)))
      (log-message "Uploading file from client to " fileloc)
      ;; 返回文件路径
      fileloc
      ))

  ;; 接收上传的数据
  (^void uploadChunk [this ^String location ^ByteBuffer chunk]
    (mark! nimbus:num-uploadChunk-calls)
    (check-authorization! nimbus nil nil "fileUpload")
    ;; nimbus uploaders是一个TimeCacheMap，
    (let [uploaders (:uploaders nimbus)
          ^WritableByteChannel channel (.get uploaders location)]
      (when-not channel
        (throw (RuntimeException.
                "File for that location does not exist (or timed out)")))
      ;; 用存储的FileOutputStream写数据
      (.write channel chunk)
      ;; 重置超时时间
      (.put uploaders location channel)
      ))

  ;; 文件上传结束 
  (^void finishFileUpload [this ^String location]
    (mark! nimbus:num-finishFileUpload-calls)
    (check-authorization! nimbus nil nil "fileUpload")
    (let [uploaders (:uploaders nimbus)
          ^WritableByteChannel channel (.get uploaders location)]
      (when-not channel
        (throw (RuntimeException.
                "File for that location does not exist (or timed out)")))
      ;; 传输完毕，关闭FileOutputStream
      (.close channel)
      (log-message "Finished uploading file from client: " location)
      ;; 从uploaders中删掉key
      (.remove uploaders location)
      ))
```

上面可以看到，提交一个topology时，首先是通过thrift接口把jar文件上传到了nimbus服务器，对于文件的存储，storm自己实现了一个BlobStore用来保存二进制数据信息。文件上传完成后，通过beginFileUpload获得的文件路径会作为一个参数传递给submitTopology接口，nimbus根据文件路径读入数据，生成jar-key并写入BlobStore结构中，接着被分配执行这个topology的supervisor再根据key从BlobStore中拉取jar文件并执行。下面是看看nimbus服务端的submitToplogy接口对传递的文件路径是如何处理的：

```clojure
(reify Nimbus$Iface
  (^void submitTopologyWithOpts
    [this ^String storm-name ^String uploadedJarLocation ^String serializedConf ^StormTopology topology
     ^SubmitOptions submitOptions]
    (try
      ; 根据提交的topology_name生成一个唯一的storm-id
      ; Example: [storm-name]-3-1430453760
      (let [storm-id (str storm-name "-" @(:submitted-count nimbus) "-" (Time/currentTimeSecs))
            ...
            total-storm-conf (merge conf storm-conf)
            topology (normalize-topology total-storm-conf topology)
            storm-cluster-state (:storm-cluster-state nimbus)]
        ...
        (locking (:submit-lock nimbus)
          ...
          (log-message "uploadedJar " uploadedJarLocation)
          ; 设置BlobStore
          (setup-storm-code nimbus conf storm-id uploadedJarLocation total-storm-conf topology)
          ; 等待数据分发到集群完成，协调确认通过zookeeper实现
          (wait-for-desired-code-replication nimbus total-storm-conf storm-id)
          ...)
      (catch Throwable e
        (log-warn-error e "Topology submission exception. (topology name='" storm-name "')")
        (throw e))))

(defn- setup-storm-code [nimbus conf storm-id tmp-jar-location storm-conf topology]
  (let [subject (get-subject)
        storm-cluster-state (:storm-cluster-state nimbus)
        blob-store (:blob-store nimbus)
        ; 生成jar-key，supervisor会根据这个key拉取文件
        ; Example: [topology-name]-0-1430453760-stromjar.jar
        jar-key (ConfigUtils/masterStormJarKey storm-id)
        code-key (ConfigUtils/masterStormCodeKey storm-id)
        conf-key (ConfigUtils/masterStormConfKey storm-id)
        nimbus-host-port-info (:nimbus-host-port-info nimbus)]
    (when tmp-jar-location  ;;in local mode there is no jar
      ; 创建Blob，这里的blob-store是NimbusBlobStore实现
      (.createBlob blob-store jar-key (FileInputStream. tmp-jar-location) (SettableBlobMeta. BlobStoreAclHandler/DEFAULT) subject)
      ...)))
```

代码执行到这里，nimbus上的BlobStore中已经有了jar文件的信息，这时候只需要执行这个topology的supervisors就可以通过相应的thrift接口来拉取jar包即可。忽略其中的消息传递机制，简单分析一下文件拉取部分的代码实现：

```clojure
; file: daemon/supervisor.clj

(defmethod download-storm-code
  :distributed [conf storm-id master-code-dir localizer]
  (let [tmproot (str (ConfigUtils/supervisorTmpDir conf) Utils/FILE_PATH_SEPARATOR (Utils/uuid))
        stormroot (ConfigUtils/supervisorStormDistRoot conf storm-id)
        ;; blobstore是一个blobclent，可以像操作本地blob一样通过thrift接口操作远程blobstore
        blobstore (Utils/getClientBlobStoreForSupervisor conf)]
    (FileUtils/forceMkdir (File. tmproot))
    ;; ConfigUtils/masterStormJarKey函数返回结果即是nimbus中的jar-key,这里通过jar-key获取到文件内容
    (Utils/downloadResourcesAsSupervisor (ConfigUtils/masterStormJarKey storm-id)
      (ConfigUtils/supervisorStormJarPath tmproot) blobstore)
    (.shutdown blobstore)
    ;; 将jar包解压到temproot目录中
    (Utils/extractDirFromJar (ConfigUtils/supervisorStormJarPath tmproot) ConfigUtils/RESOURCES_SUBDIR tmproot)
    ;; 下载topology中配置的其他的数据
    (download-blobs-for-topology! conf (ConfigUtils/supervisorStormConfPath tmproot) localizer
      tmproot)
    ...))
```

到此storm如何分发jar文件的流程基本分析完，其中TimeCacheMap和BlobStore的实现没有详细说明，实际上是对Map结构的扩展，感兴趣可以看看，另外supervisor slots的协调确认机制接下来再详细分析。
