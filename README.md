# ProxyClient
[![Build Status](https://travis-ci.org/GameXG/ProxyClient.svg?branch=master)](https://travis-ci.org/GameXG/ProxyClient)

golang 代理客户端，和 net 标准库一致的 API 。
支持嵌套代理，支持 socks4、socks4a、socks5、http、https、ss 代理协议。


``` go

package main

import (
	"fmt"
	"github.com/gamexg/proxyclient"
	"io"
	"io/ioutil"
)

func main() {
	p, err := proxyclient.NewProxyClient("socks5://127.0.0.1:5556?upProxy=https://145.2.1.3:8080")
	if err != nil {
		panic("创建代理客户端错误")
	}

	c, err := p.Dial("tcp", "www.google.com:80")
	if err != nil {
		panic("连接错误")
	}

	io.WriteString(c, "GET / HTTP/1.0\r\nHOST:www.google.com\r\n\r\n")
	b, err := ioutil.ReadAll(c)
	if err != nil {
		panic("读错误")
	}
	fmt.Print(string(b))
}

```


API
``` go

// 连接
type Conn interface {
	net.Conn
}

// 表示 TCP 连接
// 提供 net.TcpConn 全部的方法，但是部分方法由于代理协议的限制可能不能获得正确的结果。例如：LocalAddr 、RemoteAddr 方法不被很多代理协议支持。
type TCPConn interface {
	Conn

	/*
		SetLinger设定当连接中仍有数据等待发送或接受时的Close方法的行为。
		如果sec < 0（默认），Close方法立即返回，操作系统停止后台数据发送；如果 sec == 0，Close立刻返回，操作系统丢弃任何未发送或未接收的数据；如果sec > 0，Close方法阻塞最多sec秒，等待数据发送或者接收，在一些操作系统中，在超时后，任何未发送的数据会被丢弃。
	*/
	SetLinger(sec int) error

	// SetNoDelay设定操作系统是否应该延迟数据包传递，以便发送更少的数据包（Nagle's算法）。默认为真，即数据应该在Write方法后立刻发送。
	SetNoDelay(noDelay bool) error

	//SetReadBuffer设置该连接的系统接收缓冲
	SetReadBuffer(bytes int) error

	//SetWriteBuffer设置该连接的系统发送缓冲
	SetWriteBuffer(bytes int) error
}

type ProxyTCPConn interface {
	TCPConn
	ProxyClient() ProxyClient // 获得所属的代理
}

// 表示 UDP 连接
type UDPConn interface {
	Conn
}

type ProxyUDPConn interface {
	UDPConn
	ProxyClient() ProxyClient // 获得所属的代理
}
// 仿 net 库接口的代理客户端
// 支持级联代理功能，可以通过 SetUpProxy 设置上级代理。
type ProxyClient interface {
	// 返回本代理的上层级联代理
	UpProxy() ProxyClient
	// 设置本代理的上层代理
	SetUpProxy(upProxy ProxyClient) error

	// Dial 在网络network上连接地址address，并返回一个Conn接口。可用的网络类型有：
	// "tcp"、"tcp4"、"tcp6"、"udp"、"udp4"、"udp6"
	// 对TCP和UDP网络，地址格式是host:port或[host]:port，参见函数JoinHostPort和SplitHostPort。
	// 如果代理服务器支持远端DNS解析，那么会使用远端DNS解析。
	Dial(network, address string) (net.Conn, error)
	DialTimeout(network, address string, timeout time.Duration) (net.Conn, error)
	// DialTCP在网络协议net上连接本地地址laddr和远端地址raddr。net必须是"tcp"、"tcp4"、"tcp6"；如果laddr不是nil，将使用它作为本地地址，否则自动选择一个本地地址。
	// 由于 net.TCPAddr 内部保存的是IP地址及端口，所以使用本函数无法使用远端DNS解析，要想使用远端DNS解析，请使用 Dial 或 DialTCPSAddr 函数。
	DialTCP(net string, laddr, raddr *net.TCPAddr) (net.Conn, error)
	// DialTCPSAddr 同 DialTCP 函数，主要区别是如果代理支持远端dns解析，那么会使用远端dns解析。
	DialTCPSAddr(network string, raddr string) (ProxyTCPConn, error)
	// DialTCPSAddrTimeout 同 DialTCPSAddr 函数，增加了超时功能
	DialTCPSAddrTimeout(network string, raddr string, timeour time.Duration) (ProxyTCPConn, error)
	//ListenTCP在本地TCP地址laddr上声明并返回一个*TCPListener，net参数必须是"tcp"、"tcp4"、"tcp6"，如果laddr的端口字段为0，函数将选择一个当前可用的端口，可以用Listener的Addr方法获得该端口。
	//ListenTCP(net string, laddr *TCPAddr) (*TCPListener, error)
	//DialTCP在网络协议net上连接本地地址laddr和远端地址raddr。net必须是"udp"、"udp4"、"udp6"；如果laddr不是nil，将使用它作为本地地址，否则自动选择一个本地地址。
	DialUDP(net string, laddr, raddr *net.UDPAddr) (net.Conn, error)

	// 获得 Proxy 代理地址的 Query
	// 为了大小写兼容，key全部是转换成小写的。
	GetProxyAddrQuery() map[string][]string
}

// 创建代理客户端
// http 代理 http://123.123.123.123:8088
// https 代理 https://123.123.123.123:8088
// socks4 代理 socks4://123.123.123.123:5050  socks4 协议不支持远端 dns 解析
// socks4a 代理 socks4a://123.123.123.123:5050
// socks5 代理 socks5://123.123.123.123:5050?upProxy=http://145.2.1.3:8080
// ss 代理 ss://method:passowd@123.123.123:5050
// 直连 direct://0.0.0.0:0000/?LocalAddr=123.123.123.123:0
func NewProxyClient(addr string) (ProxyClient, error)

```

## 感谢

* ss 协议使用的是 [ss-go](https://github.com/shadowsocks/shadowsocks-go)
