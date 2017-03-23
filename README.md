# rpc2    [![GoDoc](https://godoc.org/github.com/tsuna/gohbase?status.png)](https://godoc.org/github.com/henrylee2cn/rpc2)

RPC2 is upgraded version RPC that based on the standard package.

It's added router Group, Plugin, InvokerSelector and customized ServiceMethod. 

![rpc2_server](https://github.com/henrylee2cn/rpc2/raw/master/doc/rpc2_server.png)

# Demo

```
package main

import (
    "errors"
    "log"
    "net/rpc"
    "strconv"
    "time"

    "github.com/henrylee2cn/rpc2"
    "github.com/henrylee2cn/rpc2/invokerselector"
    "github.com/henrylee2cn/rpc2/plugin"
)

type Worker struct{}

func (*Worker) Todo1(arg string, reply *string) error {
    rpc2.Log.Info("[server] Worker.Todo1: do job:", arg)
    *reply = "OK: " + arg
    return nil
}

func (*Worker) Todo2(arg string, reply *string) error {
    rpc2.Log.Info("[server] Worker.Todo2: do job:", arg)
    *reply = "OK: " + arg
    return nil
}

type serverPlugin struct{}

func (t *serverPlugin) Name() string {
    return "server_plugin"
}

func (t *serverPlugin) PostReadRequestHeader(ctx *rpc2.Context) error {
    rpc2.Log.Infof("serverPlugin.PostReadRequestHeader -> query: %#v", ctx.Query())
    return nil
}

type clientPlugin struct{}

func (t *clientPlugin) Name() string {
    return "client_plugin"
}

func (t *clientPlugin) PostReadResponseHeader(resp *rpc.Response) error {
    rpc2.Log.Infof("clientPlugin.PostReadResponseHeader -> resp: %v", resp)
    return nil
}

const (
    __token__ = "1234567890"
    __tag__   = "basic"
)

func checkAuthorization(serviceMethod, tag, token string) error {
    if serviceMethod != "/test/1.0.work/todo1" {
        return nil
    }
    if __token__ == token && __tag__ == tag {
        return nil
    }
    return errors.New("Illegal request!")
}

// rpc2
func main() {
    // server
    server := rpc2.NewServer(rpc2.Server{
        RouterPrintable:   true,
        ServiceMethodFunc: rpc2.NewURLServiceMethod,
    })

    // ip filter
    ipwl := plugin.NewIPWhitelistPlugin()
    ipwl.Allow("127.0.0.1")
    server.PluginContainer.Add(ipwl)

    // authorization
    group, err := server.Group(
        "test",
        plugin.NewServerAuthorizationPlugin(checkAuthorization),
        new(serverPlugin),
    )
    if err != nil {
        panic(err)
    }

    err = group.RegisterName("1.0.work", new(Worker))
    if err != nil {
        panic(err)
    }

    go server.Serve("tcp", "0.0.0.0:8080")
    time.Sleep(2e9)

    // client
    client := rpc2.NewClient(
        rpc2.Client{
            FailMode: rpc2.Failtry,
        },
        &invokerselector.DirectInvokerSelector{
            Network: "tcp",
            Address: "127.0.0.1:8080",
        },
    )

    client.PluginContainer.Add(
        plugin.NewClientAuthorizationPlugin(server.ServiceMethodFunc, __tag__, __token__),
        new(clientPlugin),
    )

    N := 100
    bad := 0
    good := 0
    mapChan := make(chan int, N)
    t1 := time.Now()
    for i := 0; i < N; i++ {
        go func(i int) {
            var reply = new(string)
            e := client.Call("/test/1.0.work/todo1?key=henrylee2cn", strconv.Itoa(i), reply)
            log.Println(i, *reply, e)
            if e != nil {
                mapChan <- 0
            } else {
                mapChan <- 1
            }
        }(i)
    }
    for i := 0; i < N; i++ {
        if r := <-mapChan; r == 0 {
            bad++
        } else {
            good++
        }
    }
    client.Close()
    server.Close()
    log.Println("cost time:", time.Now().Sub(t1))
    log.Println("success rate:", float64(good)/float64(good+bad)*100, "%")
}

```
