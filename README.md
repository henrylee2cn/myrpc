# rpc2    [![GoDoc](https://godoc.org/github.com/tsuna/gohbase?status.png)](https://godoc.org/github.com/henrylee2cn/rpc2)

RPC2 is modified version that based on the standard package, 42% performance increase.

Added router group, middleware and header information in an form of 'URL Query'. 

# usage

```
package main

import (
    "log"
    "time"
    "github.com/henrylee2cn/rpc2"
)

type Worker struct {
    Name string
}

func NewWorker() *Worker {
    return &Worker{"test"}
}

func (w *Worker) DoJob(task string, reply *string) error {
    *reply = "OK"
    return nil
}

func main() {
    // server
    server := rpc2.NewDefaultServer()
    // server.IP().Allow("127.0.0.1")
    err := server.Register(NewWorker())
    if err != nil {
        panic(err)
    }
    go server.ListenTCP("0.0.0.0:8080")
    time.Sleep(2e9)

    // client
    client := rpc2.NewClient("127.0.0.1:8080", nil)
    var reply = new(string)
    e := client.Call("Worker.DoJob", "henrylee2cn", reply)
    log.Println(*reply, e)
    e := client.Call("Worker.DoJob", "henrylee2cn--rpc2", reply)
    log.Println(*reply, e)
    client.Close()
}

```
