# Tesla hacking

OTA更新解决



MCU：Main cypher unit

- CID (Central Information Display) – the large console at the centre of the dash
- IC (Instrument Cluster)
- Conventional CAN connected ECUs
- ECU - Electronic Control Unit
- CAN - Controller Area Network
- Nvidia VCM (Visual Computer Module); 

https://www.pentestpartners.com/security-blog/reverse-engineering-tesla-hardware/

https://www.youtube.com/watch?v=0w8J9bmCI54

Wifi SSID: WIFI 名字





POC，全称”Proof of Concept”，中文“概念验证”，常指一段漏洞证明的代码。

EXP，全称”Exploit”，中文“利用”，指利用系统漏洞进行攻击的动作。

Payload，中文“有效载荷”，指成功exploit之后，真正在目标系统执行的代码或指令。

Shellcode，简单翻译“shell代码”，是Payload的一种，由于其建立正向/反向shell而得名。





### exploit 1

map=0后 if 就不进入

**用 non-shifted array 做shiftcount有memory corrupt，但用了以后会奔溃，所以用shifted array 做unshifrcount**



QTWebkit 漏洞 2.2 版本 JSArray::sort

https://github.com/NNemec/qtwebkit-2.2/blob/5fc7dcbe084a8ee8beda11b92dc2111be1285434/Source/JavaScriptCore/runtime/JSArray.cpp



73 in JSValue.h tag payload



**将storage里的内容更改**

### exploit 2

source type 源的类型



CVE-2011-3928

static_cast\<type\>(exp) 没运行时类型检查保证安全的强制转换。不能去掉const属性。Element* -> HTMLInputElement进行**下行**转换（把基类指针或引用转换成派生类表示）时，由于没有动态类型检查，所以是**不安全**的。



**看内存，从内存中获得JSCell的地址，**



## 权限

AppArmor是一个高效和易于使用的Linux系统安全应用程序

CID：连接标识符？？

CVE-2013-6282：**获得linux root权限，关闭apparmor，可以随意读写内存**

https://blog.csdn.net/py_panyu/article/details/46439999



### IC

轮换密钥在CID

### Parrot

https://keenlab.tencent.com/en/2020/01/02/exploiting-wifi-stack-on-tesla-model-s/

nc：netcat建立连接

### Gateway

1. 

Reversing bianry file "gw-diag", which is used for diagnosing gateway.

printf "\x12\x01" | socat - udp:gw:3500

把\x12\x01用udp传入gw的IP的3500端口

得到后门端口23

2. 

security token 烧录在firmware 且static, 通过reverse firmware 获得token。（如何反编译的？？）

nc gw 23 连接， 输入token进行连接。





## ECU programming

0x40000000 = 1GB

FatFs

绕过update验证















