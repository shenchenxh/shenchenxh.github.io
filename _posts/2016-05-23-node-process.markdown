---
layout: post
title:  "node多进程"
date:   2016-05-23 14:48:00 +0800
categories: node
---
本文源于《深入浅出 nodejs》
为了解决高并发的问题，node采用基于事件驱动的服务模型。

### node的多进程架构
目的：充分利用CPU资源。

* Master-Worker模式

主从模式。主进程负责调度和管理工作进程，工作进程负责具体的业务处理。

worker.js

```
var http = requier('http');
http.createServer(function() {
	res.write(200, {'Content-Type': 'text/plain'});
	res.end('Hello World\n');
}).listen(Math.round((1 + Math.random()) * 1000), '127.0.0.1');
```

master.js

```
var fork = require('child_process').fork;
var cpus = require('os').cpus();
for(var i = 0; i < cups.length; i++) {
	fork('./worker.js');
}
```

* 进程间通信

parent.js

```
var cp = require('child_process');
var n = cp.fork(__dirname + './sub.js');

n.on('message', function(m) {
	console.log('parent got message:', m);
});

n.send({hello: "world"});
```

sub.js

```
process.on('message', function(m) {
	console.log('child got message:', m);
});

process.send({foo: 'bar'});
```

* 句柄传递

服务器需要监听相同的端口：主进程收到socket请求后，将这个socket直接发送给工作进程。

1. 单个子进程

	parent.js

		var child = require('child_process').fork('child.js');
		var server = require('net').createServer();
		server.on('connection', function(socket) {
			socket.end('handled by parent')
		});
		server.listen(1337, function() {
			child.send('server', server);
		});

	child.js

		process.on('message', function(m, server) {
			if(m === 'server') {
				server.on('connection', function(socket) {
					socket.end('handled by child\n');
				})
			}
		});

2. 多个子进程

	parent.js

		var cp = require('child_process');
		var child1 = cp.fork('child.js');
		var child2 = cp.fork('child.js');

		var server = require('net').createServer();
		server.on('connection', function(socket) {
			socket.end('handled by parent')
		});
		server.listen(1337, function() {
			child1.send('server', server);
			child2.send('server', server);

			server.close();
		});

	child.js

		var http = require('http');
		var server = http.createServer(function(req, res) {
			res.writeHead(200, {'Content-Type': 'text/plain'});
			res.end('handled by child, pid is ' + process.pid + '\n');
		});

		process.on('message', function(m, tcp) {
			if(m === 'server') {
				tcp.on('connection', function(socket) {
					server.emit('connection', socket);
				})
			}
		});
