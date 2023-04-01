---
title: TinyKV Project 1 - Standalone KV
date: 2021-11-18 15:36:49
tags: 
    - Go
categories:
- Go
---

## 任务目的

熟悉Go语法，熟悉面向对象程序设计。

## 任务说明

Project1需要实现一个单节点、非分布式的K/V存储gRPC服务，本质上是对BadgerDB进行封装，以实现Put/Delete/Get/Scan四种基本操作，以及对Column Family的支持。

该Project大致分为两部分：

1. StandAloneStorage类的实现。StandAloneStorage类实现了Storage接口，Storage接口提供了用于操作K/V数据库的Start、Stop、Write、Reader四个方法，对于StandAlone的数据库来说，这四个方法就是对底层的BadgerDB进行读写。
2. Raw API的实现。Raw API为服务器对外暴露的API，用于接收请求，通过调用StandAloneStorage类的方法处理请求，然后返回响应。Raw API需要实现前述的四种基本操作。



上述调用关系如下图所示：

<img src="1-1.png" width=500>

## 任务实现

### StandAloneStorage类的实现

StandAloneStorage类的实现基本不需要直接使用BadgerDB的方法，因为engine_util包中已经封装了许多操作BadgerDB的方法，且支持了Column Family，直接调用即可。

#### func NewStandAloneStorage

该函数不是StandAloneStorage类的方法，接收config，返回一个新创建的StandAloneStorage指针。由于engines.go文件中封装的方法都以Engines类作为接收者，我们需要在创建StandAloneStorage的时候创建一个Engines实例。

```go
type StandAloneStorage struct {
    // Your Data Here (1).
    en  *engine_util.Engines
}

func NewStandAloneStorage(conf *config.Config) *StandAloneStorage {
    // Your Code Here (1).
    kvdb := engine_util.CreateDB(conf.DBPath, false)
    raftdb := engine_util.CreateDB(conf.DBPath+"/raft", true)
    return &StandAloneStorage{engine_util.NewEngines(kvdb, raftdb, conf.DBPath, conf.DBPath+"/raft")}
}
```

Raft Engine在Project1中暂时不需要用到，但如果设为nil，自带的Engines.Close方法会尝试调用nil.Close()，引发错误，因此必须进行创建。

#### func Start

根据Badger官方文档，一个Badger数据库通过Open方法打开后即可使用，最后再通过Close方法关闭。NewStandAloneStorage函数中调用的CreateDB函数已经调用了Open方法，此时数据库已经可以使用，因此Start方法无需执行任何操作。

```go
func (s *StandAloneStorage) Start() error {
    // Your Code Here (1).
    return nil
}
```

#### func Stop

Engines.Destroy一方面调用Engines.Close，关闭两个BadgerDB，另一方面删除目录。

```go
func (s *StandAloneStorage) Stop() error {
    // Your Code Here (1).
    return s.en.Destroy()
}
```

#### func Write

Write方法接收一个名为batch，元素类型为storage.Modify的切片。Modify类包含一个接口成员Data，Data可以为Put类型或者Delete类型，分别对应增加和删除操作。Put类型包含Key、Value和Cf三个成员，而Delete类型没有Value成员。

engine_util包中的WriteBatch类提供了写入KV对到数据库的方法WriteToDB。对于Write方法的实现，我们遍历Modify切片，对于每个修改项利用WriteBatch.SetCF方法将其附加到WriteBatch的底层切片末尾（类型为badger.Entry），最后再调用WriteToDB进行写入。

```go
func (s *StandAloneStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
    // Your Code Here (1).
    var wb engine_util.WriteBatch
    for _, m := range batch {
        wb.SetCF(m.Cf(), m.Key(), m.Value())
    }
    err := wb.WriteToDB(s.en.Kv)

    return err
}
```

当Modify.Data的类型为Delete时，没有Value成员，m.Value()会返回nil，在badger.Entry当中，Value成员为nil时也表示删除，所以上述代码能够正确处理添加和删除两种操作。

#### func Reader

Reader方法需返回一个实现StorageReader接口的类，该接口包含GetCF、IterCF和Close三个方法。此处我定义了一个名为Reader的类实现该接口。

engine_util包中的GetCF函数和NewCFIterator函数可用于实现前两个方法，这两个函数分别需要传入badger指针和badger.Txn指针，因此，我们的自定义Reader类，第一，需要包含StandAloneStorage成员，以访问badger；第二，需要包含badger.Txn成员，这个成员在IterCF方法中通过badger.NewTransaction函数进行创建。此外，为了在Close方法中关闭迭代器，我们还需要包含迭代器成员。综上所述，Reader类和Reader方法定义如下：

```go
type Reader struct {
    s   *StandAloneStorage
    txn *badger.Txn
    it  *engine_util.BadgerIterator
}

func (s *StandAloneStorage) Reader(ctx *kvrpcpb.Context) (storage.StorageReader, error) {
    // Your Code Here (1).
    return &Reader{s, nil, nil}, nil
}
```

GetCF方法和IterCF方法的实现参照前文。对于Close方法，需要执行关闭迭代器和丢弃txn两个操作。

```go
func (r *Reader) GetCF(cf string, key []byte) ([]byte, error) {
    val, _ := engine_util.GetCF(r.s.en.Kv, cf, key)
    return val, nil
}

func (r *Reader) IterCF(cf string) engine_util.DBIterator {
    r.txn = r.s.en.Kv.NewTransaction(false)
    r.it = engine_util.NewCFIterator(cf, r.txn)
    return r.it
}

func (r *Reader) Close() {
    if r.it != nil {
        r.it.Close()
    }
    if r.txn != nil {
        r.txn.Discard()
    }
}
```

### Raw API的实现

#### func RawGet

调用Reader.GetCF即可。需要注意的点是resp.NotFound成员必须进行赋值。

```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error) {
    // Your Code Here (1).
    reader, err := server.storage.Reader(nil)
    if err != nil {
        return nil, err
    }
    defer reader.Close()

    val, _ := reader.GetCF(req.GetCf(), req.GetKey())
    resp := new(kvrpcpb.RawGetResponse)
    if val == nil {
        resp.NotFound = true
    } else {
        resp.Value = val
        resp.NotFound = false
    }

    return resp, nil
}
```

#### func RawPut

RawPutRequest仅包含一个请求，因此只需要创建一个长度为1的Modify切片放入请求，然后调用Write。

```go
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error) {
    // Your Code Here (1).
    // Hint: Consider using Storage.Modify to store data to be modified
    batch := make([]storage.Modify, 1)
    batch[0].Data = storage.Put{Key: req.GetKey(), Value: req.GetValue(), Cf: req.GetCf()}

    err := server.storage.Write(nil, batch)
    if err != nil {
        return nil, err
    }

    return nil, nil
}
```

#### func RawDelete

同RawPut。

```go
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error) {
    // Your Code Here (1).
    // Hint: Consider using Storage.Modify to store data to be deleted
    batch := make([]storage.Modify, 1)
    batch[0].Data = storage.Delete{Key: req.GetKey(), Cf: req.GetCf()}

    err := server.storage.Write(nil, batch)
    if err != nil {
        return nil, err
    }

    return nil, nil
}
```

#### func RawScan

流程如下：

1. 获得Reader
2. 调用Reader.IterCF获得迭代器it
3. 将迭代器it定位至req.StartKey处
4. 创建切片，循环读取KV对放到切片末尾，直到KV对数量达到req.Limit，或者迭代器失效
5. 返回响应

```go
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
    // Your Code Here (1).
    // Hint: Consider using reader.IterCF
    reader, err := server.storage.Reader(nil)
    if err != nil {
        return nil, err
    }
    defer reader.Close()

    it := reader.IterCF(req.GetCf())
    it.Seek(req.StartKey)
    kvs := make([]*kvrpcpb.KvPair, 0)
    for i := 0; i < int(req.GetLimit()) && it.Valid(); i++ {
        val, _ := it.Item().Value()
        var Kv kvrpcpb.KvPair
        if val != nil {
            Kv = kvrpcpb.KvPair{Key: it.Item().Key(), Value: val}
        } else {
            Kv = kvrpcpb.KvPair{Key: it.Item().Key()}
        }
        kvs = append(kvs, &Kv)
        it.Next()
    }

    return &kvrpcpb.RawScanResponse{Kvs: kvs}, nil
}
```

容易出错的两个地方：一是没有reader没有关闭，二是迭代器没有先通过Seek方法定位到StartKey处。

## 错误与调试

### Engines的RaftDB留空

导致Destroy Engines时无法访问nil.Close()。

<img src="1-2.png">

### 没有设置RawGetResponse.NotFound

<img src="1-3.png" width=500>

### RawGet的返回值问题

当key not found时，RawGet算作正常返回，不应该返回error。

<img src="1-4.png" width=500>

### 未判断迭代器是否有效

<img src="1-5.png">

### 未将迭代器Seek到StartKey位置

<img src="1-6.png" width=500>
