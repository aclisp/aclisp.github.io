# 一个限流的 HTTPClient

> A buffered channel can be used like a semaphore, for instance to limit throughput. In this example, incoming requests are passed to handle, which sends a value into the channel, processes the request, and then receives a value from the channel to ready the “semaphore” for the next consumer. The capacity of the channel buffer limits the number of simultaneous calls to process.

利用 [channel](https://golang.org/doc/effective_go.html#channels) 限制异步函数的最大并发。 

## 创建超时可控的 HTTPClient

```
const (
	algoDumpMaxOutstanding = 100
)

var (
	algoDumpSemaphore  = make(chan int, algoDumpMaxOutstanding)
	algoDumpHTTPClient = &http.Client{
		Transport: &http.Transport{
			DialContext: (&net.Dialer{
				Timeout: 500 * time.Millisecond,
			}).DialContext,
			MaxIdleConns:        algoDumpMaxOutstanding,
			MaxIdleConnsPerHost: algoDumpMaxOutstanding,
			IdleConnTimeout:     10 * time.Minute,
		},
		Timeout: 100 * time.Millisecond,
	}
)
```

建立连接是异步的，超时仍然由 `Client.Timeout` 控制。

## HTTPClient 连接复用

```
func algorithmRecomDumpHTTP(data *pbrrec.AlgorithmRecomDumpReq) {
	dataBytes, err := jsoniter.Marshal(data)
	if err != nil {
		_ = ymetrics.CounterAdd("algo", "algo/dump_error", 1)
		ylog.Error("can not marshal data", zap.Int64("uid", data.Uid), zap.String("trace-id", data.TraceId), zap.Error(err))
		return
	}
	req, err := http.NewRequest("POST", config.DC.RoomAlgorithmDumpURL, bytes.NewReader(dataBytes))
	if err != nil {
		_ = ymetrics.CounterAdd("algo", "algo/dump_error", 1)
		ylog.Error("can not new http request", zap.Int64("uid", data.Uid), zap.String("trace-id", data.TraceId), zap.Error(err))
		return
	}
	req.Header.Set("Content-Type", "application/json")
	resp, err := algoHTTPClient.Do(req)
	if err != nil {
		_ = ymetrics.CounterAdd("algo", "algo/dump_error", 1)
		ylog.Warn("can not do http request", zap.Int64("uid", data.Uid), zap.String("trace-id", data.TraceId), zap.Error(err))
		return
	}
	_, _ = io.Copy(ioutil.Discard, resp.Body)
	_ = resp.Body.Close()
}
```

如果在读 `resp.Body` 的过程中超时，则会关闭当前连接。关键代码在 `net/http` 的

* type `cancelTimerBody` struct
* type `bodyEOFSignal` struct
  
连接复用的关键代码在 `persistConn.readLoop` 。如果该协程不退出，则相应的连接一直存在，且能被复用。

## 断路器

https://github.com/sony/gobreaker

## Use TCPDUMP to Monitor HTTP Traffic

1. To monitor HTTP traffic including request and response headers and message body:

        tcpdump -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

1. To monitor HTTP traffic including request and response headers and message body from a particular source:

        tcpdump -A -s 0 'src example.com and tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

1. To monitor HTTP traffic including request and response headers and message body from local host to local host:

        tcpdump -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' -i lo

1. To only include HTTP requests, modify “tcp port 80” to “tcp dst port 80” in above commands

1. Capture TCP packets from local host to local host

        tcpdump -i lo
