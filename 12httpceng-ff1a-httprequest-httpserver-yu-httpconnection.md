前面小节在分析 handler 时提到，handler 的读写实际是依靠 httprequest 来完成的。今天就分析 tornado 在 HTTP 这一层上的实现，类包括 HTTPRequest, HTTPServer 和 HTTPConnection.

首先，[HTTP](http://www.nowamagic.net/academy/tag/HTTP) 协议是建立在面向连接的可靠连接协议 TCP 协议之上，是应用层协议，亦即它的协议内容会涉及网络业务逻辑，而与网络连接处理等底层细节关系不大，因此今天只会提到少量 socket 内容，具体的 TCP 细节留待后面说。http 协议详情查看[RFC2616](http://tools.ietf.org/html/rfc2616)，简言之，http协议约定服务器接收到的内容应该包括三部分，首行，请求头，请求体。首行声明了请求方法，请求 uri，协议版本。请求头是一系列键值对，每行代表一个键值对，建与值间用': '分割。请求体就比较随意了，默认情况是用&连接键值对，也有可能是 RFC1867 里规定的multipart/form-data。先从 HTTPServer 说起吧。

1. HTTPServer

HTTPServer 继承于 TCPServer。它的\_\_init\_\_ 记录了连接到来时的回调函数（http 层次的回调），亦即 application 对象（它的\_\_call\_\_方法会被触发），然后就是父类的初始化了。TCP 服务器细节后面再看，简言之，它可以：它可以监听在特定地址-端口上，并每当有客户端发起连接到服务器时接收该连接，并调用方法 handle\_stream（TCP 层次的回调，这个方法总是被子类覆盖，因为只有在这里才可以实现不同应用层协议的业务逻辑）。

HTTPServer 覆盖了父类的 handler\_stream 方法，并在该方法里生成 HTTPConnection 对象就结束了。由此可知，HTTPConnection对象被构建就立即开始了 http 协议的处理，这样是合理的，因为 handle\_stream 被调用的时候肯定是新连接到来，这时缓冲区里一般有数据可读，当然可以直接读取并处理。

2. HTTPConnection

[HTTPConnection](http://www.nowamagic.net/academy/tag/HTTPConnection) 是 HTTP 协议在服务端的真正实现者，我的意思是说，对于请求的读取和解析，基本是由它（依靠HTTPHeaders）完成的。但是响应方面的处理（包括响应头，响应主体的中间处理等）则是在 RequestHandler 里的 flush 方法和 finish 方法里完成的。我把它跳过了，有兴趣的可以自己自己看吧。

HTTPConnection 里的方法，大部分都可以顾名思义，有几个方法需要注意:\_\_init\_\_, \_on\_headers, \_on\_request\_body. 其它读写之类的方法则直接对应到 IOStream 里，以后再说了。首先是\_\_init\_\_, 它也没干什么，初始化协议参数和回调函数的默认值（一般是None）。然后设定了 header\_callback，并开始读取，如下：

```
# Save stack context here, outside of any request.  This keeps
# contexts from one request from leaking into the next.
self._header_callback = stack_context.wrap(self._on_headers)
self.stream.read_until(b"\r\n\r\n", self._header_callback)

```

然后这里涉及到两个：stack\_context.wrap和stream.read\_until。

先是这个wrap函数，它就是一个装饰器，就是封装了一下，对多线程执行环境的上下文做了一些维护。

还有一个就是read\_until了，顾名思义，就是一直读取知道\r\n\r\n（这一般意味这请求头的结束），然后调用回调函数\_on\_headers（这是 IOStream 层次的回调）。具体怎么做的以后再说了，先确认是这么个功能。然后 \_on\_headers 函数在请求头结束时被调用，它的参数只有一个 data，亦即读取到的请求头字符串。首先是找到起始行：

```
eol = data.find("\r\n")
start_line = data[:eol]

```

然后是用空格分解首行来找到方法，uri和协议版本：

```
method, uri, version = start_line.split(" ")

```

接着依靠HTTPHeaders解析剩余的请求头，返回一个字典：

```
headers = httputil.HTTPHeaders.parse(data[eol:])

```

然后设定 remote\_ip（好像没什么用？）。接着创建 request 对象（这就是 RequestHandler 接收的那个 request），然后用 Content-Length 检查是否有请求体，如果没有则直接调用 HTTP 层次的回调（亦即 application 的\_\_call\_\_方法），如果有则读取指定长度的内容并跳到回调 \_on\_request\_body, 当然最终还是会调用 application 对象。在 \_on\_request\_body 方法里是调用 parse\_body\_arguments方法来完成解析主体，请求头和请求体的解析稍候再说。至此，执行流程就和 [Application对象的接口与起到的作用](http://www.nowamagic.net/academy/detail/13321021) 接上了。至于何时调用handle\_stream，后面会说到。

在看解析请求前，简单提一下 HTTPRequest。它是客户端请求的代表，它携带了所有和客户端请求的信息，因为 application 的回调\_\_call\_\_方法只接收 request 参数，当然是把所有信息包在其中。另外，由于服务器只把 request 对象暴露给 application 的回调，因此request 对象还需要提供 write，finish 方法来提供服务，其实就是对 HTTPConnection 对象的封装调用。其它也没什么了。

接下来是关于请求的解析，这和 HTTP 协议的内容密切相关。先看 httpheaders（在httputil.py里）。HTTPHeaders 继承于 dict。它的parse 是一个静态方法，接收字符串参数，内容很简单如下：

```
h = cls()
for line in headers.splitlines():
	if line:
		h.parse_line(line)
return h

```

首先创建一个自身对象，然后把字符串分解成行，对每行调用parse\_line，最后返回自身对象。以下是parse\_line：

```
if line[0].isspace():
	# continuation of a multi-line header
	new_part = ' ' + line.lstrip()
	self._as_list[self._last_key][-1] += new_part
	dict.__setitem__(self, self._last_key,
					 self[self._last_key] + new_part)
else:
	name, value = line.split(":", 1)
	self.add(name, value.strip())

```

parse\_line 方法接收一行字符串，先检查首字符是否为空格。如果不是，则用:分割该行得到键和值，保存即可。如果是空格，说明这一行的内容是从属于上一行，简单的把这一行内容附加到上次键值对的内容里。接下来是关于httputil.parse\_body\_arguments方法：

    def parse_body_arguments(content_type, body, arguments, files):
        """Parses a form request body.

        Supports ``application/x-www-form-urlencoded`` and
        ``multipart/form-data``.  The ``content_type`` parameter should be
        a string and ``body`` should be a byte string.  The ``arguments``
        and ``files`` parameters are dictionaries that will be updated
        with the parsed contents.
        """
        if content_type.startswith("application/x-www-form-urlencoded"):
            uri_arguments = parse_qs_bytes(native_str(body), keep_blank_values=True)
            for name, values in uri_arguments.items():
                if values:
                    arguments.setdefault(name, []).extend(values)
        elif content_type.startswith("multipart/form-data"):
            fields = content_type.split(";")
            for field in fields:
                k, sep, v = field.strip().partition("=")
                if k == "boundary" and v:
                    parse_multipart_form_data(utf8(v), body, arguments, files)
                    break
            else:
                gen_log.warning("Invalid multipart/form-data")


首先检查conten\_type是否以application/x-www-form-urlencoded开头，如果是，说明是用&连接的简单字符串，调用parse\_qs\_bytes（与urllib.parse.parse\_qs等效\)来进行解析。而如果是以multipart/form-data开头，则用;分解content\_type，目的是找到boundary，当找到了boundary时（这时一般content\_type也到尽头了），就调用parse\_multipart\_data对请求体进行解析。接下来看这个函数:

    def parse_multipart_form_data(boundary, data, arguments, files):
        """Parses a ``multipart/form-data`` body.

        The ``boundary`` and ``data`` parameters are both byte strings.
        The dictionaries given in the arguments and files parameters
        will be updated with the contents of the body.
        """
        # The standard allows for the boundary to be quoted in the header,
        # although it's rare (it happens at least for google app engine
        # xmpp).  I think we're also supposed to handle backslash-escapes
        # here but I'll save that until we see a client that uses them
        # in the wild.
        if boundary.startswith(b'"') and boundary.endswith(b'"'):
            boundary = boundary[1:-1]
        final_boundary_index = data.rfind(b"--" + boundary + b"--")
        if final_boundary_index == -1:
            gen_log.warning("Invalid multipart/form-data: no final boundary")
            return
        parts = data[:final_boundary_index].split(b"--" + boundary + b"\r\n")
        for part in parts:
            if not part:
                continue
            eoh = part.find(b"\r\n\r\n")
            if eoh == -1:
                gen_log.warning("multipart/form-data missing headers")
                continue
            headers = HTTPHeaders.parse(part[:eoh].decode("utf-8"))
            disp_header = headers.get("Content-Disposition", "")
            disposition, disp_params = _parse_header(disp_header)
            if disposition != "form-data" or not part.endswith(b"\r\n"):
                gen_log.warning("Invalid multipart/form-data")
                continue
            value = part[eoh + 4:-2]
            if not disp_params.get("name"):
                gen_log.warning("multipart/form-data value missing name")
                continue
            name = disp_params["name"]
            if disp_params.get("filename"):
                ctype = headers.get("Content-Type", "application/unknown")
                files.setdefault(name, []).append(HTTPFile(
                    filename=disp_params["filename"], body=value,
                    content_type=ctype))
            else:
                arguments.setdefault(name, []).append(value)


首先，保证边界 boundary 没被“”包裹。然后用b"--" + boundary + b"--"找到请求体的结束位置。然后在开头与结束位置间用分隔符b"--" + boundary + b"\r\n"把请求体分割成多个类似部分。然后对于每一部分循环进行如下处理：用b"\r\n\r\n"分隔元描述和实际内容，在元描述里可以找到 Content-Disposition, Content-type, name, filename 等信息，name 一般成为标识该部分内容的键，在字典里存储该内容。详细细节请参看[RFC1867](http://www.ietf.org/rfc/rfc1867.txt)。

至此，HTTP 层内容终于看完了。明天继续看TCP层。

