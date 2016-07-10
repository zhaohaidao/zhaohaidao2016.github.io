---
layout: post
title: 2016-07-05-python使用socket小结
---
# 背景
最近在完善分布式系统的测试系统(python实现)，需要解决跨进程通信的问题，python进程间通信是经典问题，有多种解决方法：我调研了以下三种：subprocess、multiprocess以及kv-store
## subprocess
通过pipe实现跨进程通信，但是pipe是单向的，只能由sub-process发送消息，main-process接收消息，我的需求是main-process动态发送消息控制sub-process的某些行为，因为subprocess的方案fail
## multiprocess
通过queue的方式实现跨进程通信，可是要求进程需要是原生python实现的method，不符合我的要求。
## kv-store
这个是通用解决方案，市面上一大堆kv系统：memcache、redis等等，可是在测试系统中添加一个memeche或者redis，会让测试系统变的很臃肿（我们的测试系统是分发到分布式集群的容器上执行的，所以包变大会影响部署效率）

最终决定用原生socket撸一个simple-kv，顺便复习下网络编程相关的知识

源码链接：https://github.com/zhaohaidao/Key-Value-Polyglot

server:
```python
def handle_con(conn):
    """ connection handler """
    sockfile = conn.makefile(mode="rw")
    while True:
        line = sockfile.readline()
        if line == "":
            conn.close()
            break
        parts = line.split()
        cmd = parts[0]
        if cmd == "get":
            key = parts[1]
            val = CACHE.get(key)
            if val is None:
                output(sockfile, None)
            else:
                output(sockfile, val)
        elif cmd == "set":
            key = parts[1]
            length = int(parts[2])
            val = sockfile.read(length)[:length]
            print key, val
            CACHE[key] = val

            output(sockfile, "STORED\r\n")
        sockfile.flush()

def output(sockfile, string):
    """ actually write to socket """
    sockfile.write(string)

def serve():
    """ running synchronously """
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(("127.0.0.1", port))
    sock.listen(1)

    while(1):
        conn, _ = self.sock.accept()
        thread = threading.Thread(target=handle_con, args=(conn,))
        thread.start()

    self.sock.shutdown(socket.SHUT_RDWR)
    self.sock.close()
```
client:
```python
class KVClient(object):
    """ client for kv operation """
    def __init__(self, port=11211):
        self.host = "127.0.0.1"
        self.port = int(port)

    def set(self, key, value):
        """ kv set """
        self._estab_connect()
        self._set(key, value)

    def get(self, key):
        """ do not set value to 'None' """
        self._estab_connect()
        return self._get(key)

    def _set(self, key, value):
        key = str(key)
        size = len(str(value))
        value = str(value)
        msg = ''.join(["set ", key, " ", str(size), "\r\n", str(value)])
        self._send(msg)

    def _get(self, key):
        msg = ''.join(["get ", str(key), "\r\n"])
        val = self._send(msg)
        if val == "None":
            return None
        else:
            return val

    def _send(self, msg):
        self.sock.sendall(msg)
        # TODO: 处理变长数据
        return self.sock.recv(4096)

    def _estab_connect(self):
        """ establish a new socket connection """
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((self.host, self.port))
```
