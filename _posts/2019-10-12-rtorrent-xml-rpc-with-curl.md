---
layout: post
author: Jean-Ralph Aviles
title: "rTorrent XML-RPC with cURL"
description: "Issuing rTorrent XML-RPC requests with cURL"
categories: torrents
tags: [rtorrent, curl, xmlrpc]
date: 2019-10-13T00:00:00Z
---

[![alt text](/assets/pictures/Torrent_Tyrannulet.jpg "Torrent Tyrannulet")](https://en.wikipedia.org/wiki/Torrent_tyrannulet)

# rtorrent XML-RPC with cURL
This post shows how to manually instrument [rtorrent](https://github.com/rakshasa/rtorrent)'s [XML-RPC API](https://github.com/rakshasa/rtorrent/wiki/RPC-Setup-XMLRPC). I find the usage instructions on the wiki inadequate, so I wrote my own after getting frustrated.

Ensure that rtorrent is running with `scgi_port = 127.0.0.1:5000` set in your `rtorrent.rc` or using the command line option `-o scgi_port=:5000`.

## XML-RPC
If the correct settings are used, rtorrent exposes an [XML-RPC interface](https://en.wikipedia.org/wiki/XML-RPC) that can be used to control it. An example request/response looks like this:

* Request
  ```xml
  <methodCall>                                                                                                                                 
    <methodName>d.timestamp.finished</methodName>                                                                                              
    <params>                                                                                                                                   
      <param><value><string>DD8255ECDC7CA55FB0BBF81323D87062DB1F6D1C</string></value></param>
    </params>                                                                                                                                  
  </methodCall>  
  ```
  
* Response
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <methodResponse>
  <params>
  <param><value><i8>1570426199</i8></value></param>
  </params>
  </methodResponse>
  ```
  
See the [rtorrent wiki](https://github.com/rakshasa/rtorrent/wiki/RPC-Setup-XMLRPC) for usage.

## Instrumenting API
Here are a few ways I found of intrumenting the rtorrent XML-RPC API. In order from worst to best.

### netcat
rtorrent implements XML-RPC over the [Simple Common Gateway Interface](https://en.wikipedia.org/wiki/Simple_Common_Gateway_Interface). In short, it's awful

If you read the [spec](https://python.ca/scgi/protocol.txt), it's very short, you can manually craft SCGI requests and send them to rtorrent with [netcat](https://en.wikipedia.org/wiki/Netcat).

```shell
$ cat <<EOF | xxd -r | netcat localhost 5000
00000000: 3633 3a43 4f4e 5445 4e54 5f4c 454e 4754  63:CONTENT_LENGT
00000010: 4800 3730 0053 4347 4900 3100 5245 5155  H.70.SCGI.1.REQU
00000020: 4553 545f 4d45 5448 4f44 0050 4f53 5400  EST_METHOD.POST.
00000030: 5245 5155 4553 545f 5552 4900 2f52 5043  REQUEST_URI./RPC
00000040: 3200 2c3c 6d65 7468 6f64 4361 6c6c 3e20  2.,<methodCall>
00000050: 203c 6d65 7468 6f64 4e61 6d65 3e73 7973   <methodName>sys
00000060: 7465 6d2e 6c69 7374 4d65 7468 6f64 733c  tem.listMethods<
00000070: 2f6d 6574 686f 644e 616d 653e 3c2f 6d65  /methodName></me
00000080: 7468 6f64 4361 6c6c 3e                   thodCall>
EOF
```

### cURL + nginx
The easiest approach I found uses the nginx [ngx_http_scgi_module](http://nginx.org/en/docs/http/ngx_http_scgi_module.html) to do the heavy lifting with SCGI. nginx exposes an HTTP🠖XML-RPC interface which you can use with cURL. Requests are proxied as SCGI to rtorrent.

1. [Install nginx](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
1. Configure it as follows:
   ```nginx
   http {
      server {
        listen 8888;
        location /RPC2 {
          scgi_pass localhost:5000;
          include scgi_params;
        }
      }
    }
    events {
    }

   ```
1. Start nginx
   ```bash
   $ nginx -c '/nginx.conf' -g 'daemon on;' 
   ```
1. Make your request
   ```bash
   $ cat <<EOF | curl -d @/dev/stdin localhost:8888/RPC2
   <methodCall>                                                                                                                                 
     <methodName>d.timestamp.finished</methodName>                                                                                              
     <params>                                                                                                                                   
       <param><value><string>DD8255ECDC7CA55FB0BBF81323D87062DB1F6D1C</string></value></param>
     </params>                                                                                                                                  
   </methodCall>                                                                                                                                
   EOF  
   ```
   
That's it.
   
#### xmlrpc
There's a tool bundled with the [xmlrpc-c](http://xmlrpc-c.sourceforge.net/) library, `xmlrpc`, that can be used to instrument this API with nginx. It's not available in all package managers, but its source can be found on this [GitHub mirror](https://github.com/mirror/xmlrpc-c). If you can find it in your package manager or compile it yourself, I couldn't get it to build, you can use it as such.

```bash
$ xmlrpc localhost:8888 system.listMethods
```

## Sources

* [rTorrent RPC Setup XMLRPC](https://github.com/rakshasa/rtorrent/wiki/RPC-Setup-XMLRPC)
* [XML-RPC Specification](http://xmlrpc.scripting.com/spec.html)
* [SCGI Specification](https://python.ca/scgi/protocol.txt)
