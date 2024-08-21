
SipP是一个短小精悍的脚本工具，可以支持模拟客户端与服务端的模式

## About server

```bash
./sipp -sf example-server.xml -i host -p port  --max_socket 100000 -l 500000
```

## About client

```bash
./sipp proxyhost:port -sf example-client.xml  -r batchsize  -d 10000 -s caller -m totalcount
```

## About testcase

### successful call bye from client

```bash
./sipp -sf server-200-client-bye.xml -i host -p port  --max_socket 100000 -l 500000
./sipp proxyhost:port -sf client-200-bye.xml  -r 5  -d 10000 -s 20240715 -m 5
```

### successful call bye from server

```bash
./sipp -sf server-200-bye.xml -i host -p port  --max_socket 100000 -l 500000
./sipp proxyhost:port -sf client-200-server-bye.xml  -r 5  -d 10000 -s 20240715 -m 5
```

