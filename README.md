# Redis 客户端实现原理

## Redis 协议

要编写一个 Redis 客户端，首先需要了解 Redis 的通讯协议。Redis 在 TCP 端口 6379 上监听到来的连接，客户端连接到来时，Redis 服务器为此创建一个 TCP 连接。
在客户端与服务器端之间传输的每个 Redis 命令或者数据都以 \r\n 结尾。

例如执行命令 KEYS * ，只需要向服务器发送 KEYS *\r\n 即可，服务器会直接响应结果，回复的格式如下

- 用单行回复，回复的第一个字节将是“+”
- 用单行回复，回复的第一个字节将是“+”
- 整型数字，回复的第一个字节将是“:”
- 批量回复，回复的第一个字节将是“$”
- 多个批量回复，回复的第一个字节将是“*”

用 nc 测试各个结果返回的格式如下：

```bash
$ nc localhost 6379
```

1、返回错误

```bash
haha

-ERR unknown command 'haha'
```

2、操作成功

```bash
auth luobin

+OK
```

3、得到结果

```bash
get foo

$3
bar
```

4、没有得到结果

```bash
get test

$-1
```

5、结果为整型数字

```bash
hlen user

:4
```

6、批量结果

```bash
keys *

*3
$2
li
$10
string key
$4
list

```

7、多命令

```bash
multi

+OK

get foo

+QUEUED

get foo1

+QUEUED

exec

*2
$3
bar
$3
bar
```

所以 Redis 客户端实现的核心功能即将需要执行的命令依次发给服务器，服务器会按照先后顺序把结果返回给用户。
在 node_redis 这个客户端中，使用了内置的 net 模块来进行 tcp 的操作，通过 data 事件来接收消息。

## 源码分析

### 项目依赖

```json
"dependencies": {
  "double-ended-queue": "^2.1.0-0",
  "redis-commands": "^1.2.0",
  "redis-parser": "^1.3.0"
},
```

[double-ended-queue](https://github.com/petkaantonov/deque) 是一个实现双向队列的库，在 node_redis 中
用于存储发送给 redis 的命令。主要有三种类型的 Queue，command_queue、offline_queue 和
pipeline_queue。offline_queue 是在连接还没有准备好的情况下会将命令存在这里，当连接 ready 后，会将其中的
 command 全部发送给 redis 。每次发送给 redis 的命令会存放在 command_queue 中，当返回成功的时候会将命令移除。
 pipeline_queue 保存所有管道命令，发送后将命令移除。

[redis-commands](https://github.com/NodeRedis/redis-commands) 这个模块包含所有 Redis 支持的命令。

[redis-parser](https://github.com/NodeRedis/node-redis-parser) 这个模块用来解析redis返回的数据。

### 核心模块

首先看 index.js，找到 createClient 函数

```js
exports.createClient = function () {
    return new RedisClient(unifyOptions.apply(null, arguments));
};
```

可以看到这个函数返回了一个 RedisClient 的对象，并且通过 unifyOptions 函数对参数进行了处理，找到 unifyOptions 函数

```js
var unifyOptions = require('./lib/createClient');
```

找到 /lib/createClient.js

```js
module.exports = function createClient (port_arg, host_arg, options) {

    if (typeof port_arg === 'number' || typeof port_arg === 'string' && /^\d+$/.test(port_arg)) {
        // ...
    } else if (typeof port_arg === 'string' || port_arg && port_arg.url) {
        // ...
    } else if (typeof port_arg === 'object' || port_arg === undefined) {
        // ...
    }

    if (!options) {
        throw new TypeError('Unknown type of connection in createClient()');
    }

    return options;
};
```

这个函数主要对各种情况的参数进行处理，返回统一的 options。

```js
// 参数的各种情况([] 表示这个参数可以为空)
redis.createClient([options]) // 走第三条分支
redis.createClient(unix_socket[, options]) // 走第三条分支
redis.createClient(redis_url[, options]) // 走第二条分支
redis.createClient(port[, host][, options]) // 走第一条分支
```

再来看 RedisClient 函数，我在关键的参数和代码加了注释

```js
function RedisClient (options, stream) {
    // 拷贝一份 options
    options = utils.clone(options);
    // 调用EventEmitter
    EventEmitter.call(this);
    // 保存连接参数
    var cnx_options = {};
    var self = this;
    // 如果参数带了 tsl，则会内部会建立 tsl 连接
    for (var tls_option in options.tls) {
        cnx_options[tls_option] = options.tls[tls_option];
        if (tls_option === 'port' || tls_option === 'host' || tls_option === 'path' || tls_option === 'family') {
            options[tls_option] = options.tls[tls_option];
        }
    }
    // socket 对象可以由参数传入
    if (stream) {
        options.stream = stream;
        this.address = '"Private stream"';
    } else if (options.path) {
        cnx_options.path = options.path;
        this.address = options.path;
    } else {
        cnx_options.port = +options.port || 6379;
        cnx_options.host = options.host || '127.0.0.1';
        cnx_options.family = (!options.family && net.isIP(cnx_options.host)) || (options.family === 'IPv6' ? 6 : 4);
        this.address = cnx_options.host + ':' + cnx_options.port;
    }
    // 带 retry_strategy 参数则 max_attempts，retry_max_delay 参数无效
    if (typeof options.retry_strategy === 'function') {
        if ('max_attempts' in options) {
            self.warn('WARNING: You activated the retry_strategy and max_attempts at the same time. This is not possible and max_attempts will be ignored.');
            delete options.max_attempts;
        }
        if ('retry_max_delay' in options) {
            self.warn('WARNING: You activated the retry_strategy and retry_max_delay at the same time. This is not possible and retry_max_delay will be ignored.');
            delete options.retry_max_delay;
        }
    }

    this.connection_options = cnx_options;
    // connection_id 是一个内部变量，维护连接数
    this.connection_id = RedisClient.connection_id++;
    // 标记是否连接，连接是否ready的变量
    this.connected = false;
    this.ready = false;
    
    // socket_nodelay 已经 remove
    if (options.socket_nodelay === undefined) {
        options.socket_nodelay = true;
    } else if (!options.socket_nodelay) {
        self.warn(
            'socket_nodelay is deprecated and will be removed in v.3.0.0.\n' +
            'Setting socket_nodelay to false likely results in a reduced throughput. Please use .batch for pipelining instead.\n' +
            'If you are sure you rely on the NAGLE-algorithm you can activate it by calling client.stream.setNoDelay(false) instead.'
        );
    }
    
    // socket_keepalive 设置 socket keepalive
    if (options.socket_keepalive === undefined) {
        options.socket_keepalive = true;
    }
    
    // redis 命令重命名
    for (var command in options.rename_commands) {
        options.rename_commands[command.toLowerCase()] = options.rename_commands[command];
    }
    // return_buffers 为 true 时所有命令都返回 Buffer
    options.return_buffers = !!options.return_buffers;
    // detect_buffers 为 true 时可以在每一个命令上自由切换返回 String 或者 Buffer
    options.detect_buffers = !!options.detect_buffers;
    if (options.return_buffers && options.detect_buffers) {
        self.warn('WARNING: You activated return_buffers and detect_buffers at the same time. The return value is always going to be a buffer.');
        options.detect_buffers = false;
    }
    if (options.detect_buffers) {
        this.handle_reply = handle_detect_buffers_reply;
    }
    this.should_buffer = false;
    
    // max_attempts 已过时
    this.max_attempts = options.max_attempts | 0;
    if ('max_attempts' in options) {
        self.warn(
            'max_attempts is deprecated and will be removed in v.3.0.0.\n' +
            'To reduce the amount of options and the improve the reconnection handling please use the new `retry_strategy` option instead.\n' +
            'This replaces the max_attempts and retry_max_delay option.'
        );
    }
    
    // 初始化上面提到的三个 Queue
    this.command_queue = new Queue();
    this.offline_queue = new Queue();
    this.pipeline_queue = new Queue();
    
    // connect_timeout 设置 socket tomeout
    this.connect_timeout = +options.connect_timeout || 3600000; // 60 * 60 * 1000 ms
    
    // 默认情况下在连接没有 ready 的情况下会将命令保存在 offline_queue 中，ready 之后执行
    // enable_offline_queue 设置为 false 关闭此功能
    this.enable_offline_queue = options.enable_offline_queue === false ? false : true;
    
    // 设置重连延迟的最大值
    this.retry_max_delay = +options.retry_max_delay || null;
    if ('retry_max_delay' in options) {
        self.warn(
            'retry_max_delay is deprecated and will be removed in v.3.0.0.\n' +
            'To reduce the amount of options and the improve the reconnection handling please use the new `retry_strategy` option instead.\n' +
            'This replaces the max_attempts and retry_max_delay option.'
        );
    }
    
    // 初始化一堆变量
    this.initialize_retry_vars();
    this.pub_sub_mode = 0;
    this.subscription_set = {};
    this.monitoring = false;
    this.message_buffers = false;
    this.closing = false;
    this.server_info = {};
    this.auth_pass = options.auth_pass || options.password;
    this.selected_db = options.db;
    this.old_state = null;
    this.fire_strings = true;
    this.pipeline = false;
    this.sub_commands_left = 0;
    this.times_connected = 0;
    this.options = options;
    this.buffers = options.return_buffers || options.detect_buffers;
    this.reply = 'ON';
    
    // 创建 parser
    this.reply_parser = create_parser(this, options);
    // 创建连接 socket
    this.create_stream();
    
    this.on('newListener', function (event) {
        if (event === 'idle') {
            this.warn(
                'The idle event listener is deprecated and will likely be removed in v.3.0.0.\n' +
                'If you rely on this feature please open a new ticket in node_redis with your use case'
            );
        } else if (event === 'drain') {
            this.warn(
                'The drain event listener is deprecated and will be removed in v.3.0.0.\n' +
                'If you want to keep on listening to this event please listen to the stream drain event directly.'
            );
        } else if (event === 'message_buffer' || event === 'pmessage_buffer' || event === 'messageBuffer' || event === 'pmessageBuffer' && !this.buffers) {
            this.message_buffers = true;
            this.handle_reply = handle_detect_buffers_reply;
            this.reply_parser = create_parser(this);
        }
    });
}

// 继承 EventEmitter
util.inherits(RedisClient, EventEmitter);
```

主要功能就是初始化了变量，有两个关键的地方就是 create_parser 和 create_stream，前者用于处理解析 
redis 返回的数据，后者用于处理 redis 连接的各种事件（connect, data, end, timeout, drain, error, close）

```js
function create_parser (self) {
    return Parser({
        returnReply: function (data) {
            self.return_reply(data);
        },
        returnError: function (err) {
            // Return a ReplyError to indicate Redis returned an error
            self.return_error(new errorClasses.ReplyError(err));
        },
        returnFatalError: function (err) {
            // Error out all fired commands. Otherwise they might rely on faulty data. We have to reconnect to get in a working state again
            // Note: the execution order is important. First flush and emit, then create the stream
            err = new errorClasses.ReplyError(err);
            err.message += '. Please report this.';
            self.ready = false;
            self.flush_and_error({
                message: 'Fatal error encountert. Command aborted.',
                code: 'NR_FATAL'
            }, {
                error: err,
                queues: ['command_queue']
            });
            self.emit('error', err);
            self.create_stream();
        },
        returnBuffers: self.buffers || self.message_buffers,
        name: self.options.parser,
        stringNumbers: self.options.string_numbers
    });
}
```

可以看到 parser 会抛出三个事件，返回 reply、返回错误、返回致命的错误。再来看 create_stream

```js
RedisClient.prototype.create_stream = function () {
    var self = this;
    
    // 创建连接
    if (this.options.stream) {
        // 重连
        if (this.stream) {
            return;
        }
        this.stream = this.options.stream;
    } else {
        // 重连
        if (this.stream) {
            this.stream.removeAllListeners();
            this.stream.destroy();
        }
        
        // tsl 连接或者 tcp 连接
        if (this.options.tls) {
            this.stream = tls.connect(this.connection_options);
        } else {
            this.stream = net.createConnection(this.connection_options);
        }
    }
    
    // 设置 timeout，超时会进行重连操作
    if (this.options.connect_timeout) {
        this.stream.setTimeout(this.connect_timeout, function () {
            self.retry_totaltime = self.connect_timeout;
            self.connection_gone('timeout');
        });
    }

    var connect_event = this.options.tls ? 'secureConnect' : 'connect';
    
    // connect 事件
    this.stream.once(connect_event, function () {
        this.removeAllListeners('timeout');
        self.times_connected++;
        self.on_connect();
    });

    // data 事件
    this.stream.on('data', function (buffer_from_socket) {
        debug('Net read ' + self.address + ' id ' + self.connection_id); // + ': ' + buffer_from_socket.toString());
        self.reply_parser.execute(buffer_from_socket);
        self.emit_idle();
    });
    
    // error 事件
    this.stream.on('error', function (err) {
        self.on_error(err);
    });
    
    // error 事件
    this.stream.on('clientError', function (err) {
        debug('clientError occured');
        self.on_error(err);
    });
    
    // close 事件会进行重连操作
    this.stream.once('close', function (hadError) {
        self.connection_gone('close');
    });
    
    // end 事件会进行重连操作
    this.stream.once('end', function () {
        self.connection_gone('end');
    });
    
    this.stream.on('drain', function () {
        self.drain();
    });
    
    // 设置 no delay
    if (this.options.socket_nodelay) {
        this.stream.setNoDelay();
    }

    // auth 验证
    if (this.auth_pass !== undefined) {
        this.ready = true;
        this.auth(this.auth_pass);
        this.ready = false;
    }
};
```

可以看到，当 redis 返回数据的时候会调用 parse 的 execute 函数来处理数据，找到这个函数

```js
HiredisReplyParser.prototype.parseData = function () {
    try {
        return this.reader.get();
    } catch (err) {
        this.reader = new hiredis.Reader(this.options);
        this.returnFatalError(err);
    }
};

HiredisReplyParser.prototype.execute = function (data) {
    this.reader.feed(data);
    var reply = this.parseData();

    while (reply !== undefined) {
        if (reply && reply.name === 'Error') {
            this.returnError(reply);
        } else {
            this.returnReply(reply);
        }
        reply = this.parseData();
    }
};
```

execute 函数会解析数据并且根据返回的结果抛出不同的事件，与创建 paser 时的函数一一对应。
正常返回会抛出 returnReply 事件，根据 create_parser 可以看到 returnReply 事件会调用 RedisClient 的
 return_reply 函数
 
 ```js
 RedisClient.prototype.return_reply = function (reply) {
    
    // ...
    // ...
    // ...
    
    if (this.pub_sub_mode === 0) {
        normal_reply(this, reply);
    } else if (this.pub_sub_mode !== 1) {
        this.pub_sub_mode--;
        normal_reply(this, reply);
    } else if (!(reply instanceof Array) || reply.length <= 2) {
        normal_reply(this, reply);
    } else {
        return_pub_sub(this, reply);
    }
};
 ```
 
 先不管 pub sub 的情况，会继续调用 normal_reply 函数。normal_reply 会先从 command_queue 中取出 command，并且调用回调
 
 ```js
 function normal_reply (self, reply) {
    var command_obj = self.command_queue.shift();
    if (typeof command_obj.callback === 'function') {
        if (command_obj.command !== 'exec') {
            reply = self.handle_reply(reply, command_obj.command, command_obj.buffer_args);
        }
        command_obj.callback(null, reply);
    } else {
        debug('No callback for reply');
    }
}
 ```
 
 这样如何处理 redis 回复的消息就比较清楚了。再来看看如何处理 redis 连接的各种事件，先看 connect，找到 on_connect 函数
 
 ```js
 RedisClient.prototype.on_connect = function () {
    debug('Stream connected ' + this.address + ' id ' + this.connection_id);
    
    this.connected = true;
    this.ready = false;
    this.emitted_end = false;
    this.stream.setKeepAlive(this.options.socket_keepalive);
    this.stream.setTimeout(0);
    
    // 向外抛出 connect 事件
    this.emit('connect');
    this.initialize_retry_vars();
    
    if (this.options.no_ready_check) {
        this.on_ready();
    } else {
        this.ready_check();
    }
};
 ```
 
 on_connect 函数会先更新状态，然后抛出事件，设置 no_ready_check 为 true 会先测试一下连接是否成功，
 类似 ping 操作，然后调用 on_ready 函数，如果设置为 false 会直接调用 on_ready 函数
 
 ```js
 RedisClient.prototype.on_ready = function () {
    var self = this;

    debug('on_ready called ' + this.address + ' id ' + this.connection_id);
    this.ready = true;

    // ...
    // ...
    // ...

    this.send_offline_queue();
    // 向外抛出 ready 事件
    this.emit('ready');
};
 ```

on_ready 函数最核心的是会调用 send_offline_queue 函数，发送保存在 offline_queue 的命令。

```js
RedisClient.prototype.send_offline_queue = function () {
    for (var command_obj = this.offline_queue.shift(); command_obj; command_obj = this.offline_queue.shift()) {
        debug('Sending offline command: ' + command_obj.command);
        this.internal_send_command(command_obj.command, command_obj.args, command_obj.callback, command_obj.call_on_write);
    }
    this.drain();
};
```

internal_send_command 函数遍历 offline_queue，internal_send_command 函数执行所有的命令

```js
RedisClient.prototype.internal_send_command = function (command, args, callback, call_on_write) {
   
   // ...
   // ...
   // ...

    if (this.ready === false || this.stream.writable === false) {
        // Handle offline commands right away
        handle_offline_command(this, new OfflineCommand(command, args, callback, call_on_write));
        return false; // Indicate buffering
    }
    
    // ...
    // ...
    // ...

    if (big_data === false) {
        for (i = 0; i < len; i += 1) {
            arg = args_copy[i];
            command_str += '$' + Buffer.byteLength(arg) + '\r\n' + arg + '\r\n';
        }
        debug('Send ' + this.address + ' id ' + this.connection_id + ': ' + command_str);
        this.write(command_str);
    } else {
        // ...
        // ...
        // ...
    }
    
    // ...
    // ...
    // ...
};
```

internal_send_command是一个直接向 redis 发送命令的函数，其他的 API 接口也会调用此函数。从 internal_send_command 
中可以看到在 ready === false 时会调用 handle_offline_command 函数把命令保存进 offline_queue 中。关键的发送命令的
函数为 write 函数。

```js
RedisClient.prototype.write = function (data) {
    // 如果 pipeline === false，会直接调用 socket 发送数据。否则保存进 pipeline_queue
    if (this.pipeline === false) {
        this.should_buffer = !this.stream.write(data);
        return;
    }
    this.pipeline_queue.push(data);
};
```

这样发送命令的流程也已经清楚。再来看 on_error 函数

```js
RedisClient.prototype.on_error = function (err) {
    if (this.closing) {
        return;
    }

    err.message = 'Redis connection to ' + this.address + ' failed - ' + err.message;
    debug(err.message);
    this.connected = false;
    this.ready = false;

    if (!this.options.retry_strategy) {
        this.emit('error', err);
    }
    this.connection_gone('error', err);
};
```

on_error 函数会先更新状态，如果没有设置 retry_strategy 会向外抛出错误，最后调用 connection_gone 函数，end 和 close 事件也直接调用 connection_gone 函数进行重连操作

```js
RedisClient.prototype.connection_gone = function (why, error) {
    // ...
    // ...
    // ...
    this.retry_timer = setTimeout(retry_connection, this.retry_delay, this, error);
};

var retry_connection = function (self, error) {
    debug('Retrying connection...');

    // ...
    // ...
    // ...
    
    // 向外抛出 reconnecting 事件
    self.emit('reconnecting', reconnect_params);

    // ...
    // ...
    // ...
    
    // 创建连接
    self.create_stream();
    // 设置 retry_timer 为空
    self.retry_timer = null;
};
```

重连会调用 create_stream 函数重新创建一个连接。这样整个核心的代码的逻辑就比较清楚了。

备注：更友好的发送命令的函数在 /lib/individualCommands.js 文件中，包括 set，get，auth等。
