# SOCKS
[![Build Status](https://travis-ci.org/eahydra/socks.svg?branch=master)](https://travis-ci.org/eahydra/socks)  [![GoDoc](https://godoc.org/github.com/eahydra/socks?status.svg)](https://godoc.org/github.com/eahydra/socks)

SOCKS implements SOCKS4/5 Proxy Protocol and HTTP Tunnel which can help you get through firewall.
The [cmd/socksd](https://github.com/eahydra/socks/blob/master/cmd/socksd) build with package SOCKS, supports cipher connection which crypto method is rc4, des, aes-128-cfb, aes-192-cfb and aes-256-cfb, upstream which can be shadowsocks or socsk5.

# Install
Assume you have go installed, you can install from source.
```
go get github.com/eahydra/socks
```

If you want to install [cmd/socksd](https://github.com/eahydra/socks/blob/master/cmd/socksd), please read the [README.md](https://github.com/eahydra/socks/blob/master/cmd/socksd/README.md)

实现一个代理服务
在天朝做程序员比较让人蛋疼，比如你想用GOOGLE，你就很蛋疼。原因大家都懂。

然后呢，一开始自己在用GOAGENT, VPN, SSH, ShadowSocks等程序，GOAGENT和SHADOWSOCKS都是非常优秀的。而自己在很早刚开始接触计算机的时候就有想法自己写一个代理程序，因为各种各样的原因总是没去做，或者说自己的需求总是能够被满足，所以没什么动力。但是自从学GO语言后，网络程序的开发变的没有之前做C/C++时那么蛋疼了，所以试着自己写一个代理程序，然后也贯彻Eating your own dog food的实践。这里把一些心得总结记录下。GITHUB地址：https://github.com/eahydra/socks

 

SOCKS5协议
刚开始的时候，是想着做HTTP代理，但是HTTP协议比较复杂，牵扯到的概念比较多，自己又不熟悉，所以就没选中做HTTP通道方式的代理程序。而代理协议里用的最多的其实就是SOCKS协议了。SOCKS4，SOCKS5等。因为我一直在用CHROME, CHROME上的插件SwitchySharp也比较好用，也支持SOCKS5协议，更重要的是SOCKS5协议非常简单。

SOCKS5协议主要分三步，第一步就是握手协商，协商双方的验证方式。我的做法简单粗暴：因为SOCKS5规定的握手协议中，规定了最大的数据长度就是258个直接，所以我在实现的时候，直接申请258字节的BUFFER，然后读取。读取后也不对验证方式进行判断是否合法，支持之类的，直接返回0X05 0X00，也就是不需要验证。

第二步就是获取客户端发来的CMD协议。SOCKS5这里也规定了最长长度，263个字节。所以也一次申请出来以备后用。感谢GO语言封装的网络库，不管是啥域名，还是IPV4，IPV6之类的，调用net.Dial的时候传进去就好，net.Dial函数内部会自己搞定到IP的转换。然后就根据CMD里指定的目标地址链接，成功后返回对应的协议数据。目前我只支持了CONNECT，像UDP之类的就没做支持，遇到了UDP指令直接告诉客户端不支持，然后断掉链接。

第三步就是开始接受客户端发送来的数据，然后发送到远端服务器，然后从远端服务器读数据再转发到客户端。这个就简单了，直接两行代码搞定。

go io.Copy(dest, src)
io.Copy(src, dest)
 

部署
写好程序后，就得部署起来。感谢亚马逊提供了免费一年的AWS服务。怎么申请之类的自行搜索解决。GO的移植性这里就体现出来了，因为我开发是在WINDOWS下，然后部署到AWS上的UBUNTU系统上，一个go build搞定。然后程序跑起来后，直接设置SwitchySharp到AWS上，效果显著。不过中间测试的时候，发现链接到http://go-talks.appspot.com/github.com/extemporalgenome/gotalks/error-handling.slide#1 就不行了，抓包发现，从AWS上发来了很多RST，但是我的程序没有打印出对应的CLOSE信息。这个时候我就猜测应该是那个啥（你懂的）做了内容检测，解决的方法之一就是加密。所以我后面就做了加密的功能。

加密
增加加密功能就蛋疼了，因为我可能不只是要一种加密算法，可能这会想用RC4，后面就想用AES或者DES之类的。还有就是怎么把加密算法封装起来，使其对现有的代码影响较小。本来我是想自己手工封装成io.Reader和io.Writer的接口，这样的话就可以组成一个调用链，很自然的把网络数据进行加解密。然后我就去GO的标准库里找，有加密算法，RC4，DES,AES都有提供，然后有个叫crypto/cipher的包，这个包提供了俩接口，一个叫StreamRead，另外一个叫StreamWrite,这俩接口正好可以满足io.Reader和io.Write。这下子就简单多了。具体的代码如下(代码里没有做什么工厂模式之类的，因为实在是没啥必要，写的简单粗暴点也不会影响其他代码)：

复制代码
func NewCipherStream(rwc io.ReadWriteCloser, cryptMethod string, password []byte) (*CipherStream, error) {
    var stream *CipherStream
    switch cryptMethod {
    default:
        stream = &CipherStream{
            reader:      rwc,
            writeCloser: rwc,
        }

    case "rc4":
        {
            rc4CipherRead, err := rc4.NewCipher(password)
            if err != nil {
                return nil, err
            }
            rc4CipherWrite, err := rc4.NewCipher(password)
            if err != nil {
                return nil, err
            }

            stream = &CipherStream{
                reader: &cipher.StreamReader{
                    S: rc4CipherRead,
                    R: rwc,
                },
                writeCloser: &cipher.StreamWriter{
                    S: rc4CipherWrite,
                    W: rwc,
                },
            }
        }
    case "des":
        {
            block, err := des.NewCipher(password)
            if err != nil {
                return nil, err
            }
            desRead := cipher.NewCFBDecrypter(block, desIV[:])
            desWrite := cipher.NewCFBEncrypter(block, desIV[:])
            return &CipherStream{
                reader: &cipher.StreamReader{
                    S: desRead,
                    R: rwc,
                },
                writeCloser: &cipher.StreamWriter{
                    S: desWrite,
                    W: rwc,
                },
            }, nil
        }
    }
    return stream, nil
}
复制代码
如果没有指定加密方式或者是不支持的加密方式，还是沿用原有的io.Reader和io.Writer接口。这样的话就可以减少对上层的影响。

因为支持了多种加密方式，那么就得用配置文件了。这个就很简单，直接JSON格式搞定。

另外一个问题是，浏览器只认SOCKS协议，你现在加了加密，那你得对浏览器隐藏掉，然后就得支持俩模式，一个模式是本地的SOCKS运行方式，对浏览器提供的，另外一个模式就是到远端SOCKS服务的数据要加密。本来是想再写一个本地SOCKS，但是想了下，实在是毫无必要，因为这个程序的大部分功能在现有的代码基础上都能满足，那么一种实现方式就是代码拷贝出来，但是我要维护两分；另外一种方式就是在原有的代码上改。决定在现有代码上改，无非是在客户端的请求来了后，要先链接到自己的远端SOCKS服务，然后再提供数据中转服务。这里就涉及到怎么和远端服务交互的问题。

PS: 现在最新版本支持SHADOWSOCKS协议，对AES,DES等加密算法的应用做了修改。AES和DES需要的IV值会在数据头部传输，但只传输一次。同时还会对密码做相应的处理。具体看代码便可明白。

和远端SOCKS交互的解决方法
为了解决这个问题，就得需要订协议。是否有必要增加协议呢？我第一个想法是好像没必要，第二个想法是真的没必要？真的没必要！因为SOCKS5的协议足够使用，我也不想因为要链接到远端还要增加一套额外的协议，而这个协议最终也会和SOCKS5差不多，实现出来的代码又会很蛋疼。所以决定使用SOCKS5协议，做成一个调用链的形式。客户端来的握手，本地SOCKS也到远端SOCKS上进行握手。客户端发来的CMD直接转发到远端SOCKS。然后中间走加密。这下子实现就变的非常简单了。定义了一个结构体RemoteSocks，匿名组合了*CipherStream，这样就可以直接走加密了。具体定义代码如下：

type RemoteSocks struct {
    conn net.Conn
    *CipherStream
}
然后同步代码到AWS上，编译开跑，然后发现很流畅。给基友狗眼坤分享了下，遇到了BUG，在他机器上链接YOUTUBE出现链接错误。然后想大概问题在哪，开始做代码REVIEW，发现自己实现的加密部分，RC4相关的部分读和写复用了同一个加密对象，然后这个对象又不是线程安全的，问题应该是这里了，修正掉，然后测试之，OK了。

总结
总算是有了自己的代理程序，用起来很开心，因为是自己做的！Eating your own dog food很重要。自己实现一个程序的时候，可以不用一下子跨很大的步子，那样容易扯到蛋，可以一点点来。
