---
layout: post
title: 自制Java Class文件解析器之起步
date: 2019-04-05 17:03
author: zsh
catalog: true
tags:
    - Java
    - JVM
---



听说《深入理解JVM》虚拟机第三版出版了，于是我赶紧上京东下单了一本，也算是第一批阅读的用户了吧。读到第六章的时候突发奇想，能不能自己写一个简易的类文件解析器，要求也不高，只要把类文件的各个部分都能解析出来，最后按照自己喜欢的格式打印出来就可以了。而且支持的JDK版本在8以下。说干就干。



首先回顾一下书上Class文件的组成部分

![](/img/2019-04-05-classparser-1/class-structure.png)

《深入理解JVM》里提到，Class文件的结构既不像XML文件那样表意充分，也不想JSON那样言简意赅，它只是纯的二进制文件，因此什么位置是什么数据会非常精确。我们可以用**xxd**命令查看一下Class文件内容

```bash
xxd TestClass.class
```

输出内容

```
00000000: cafe babe 0000 0034 0022 0a00 0900 1a08  .......4."......
00000010: 001b 0900 0800 1c09 0008 001d 0500 0000  ................
00000020: 0000 0000 0507 001e 0700 1f07 0020 0100  ............. ..
00000030: 046e 616d 6501 0012 4c6a 6176 612f 6c61  .name...Ljava/la
00000040: 6e67 2f53 7472 696e 673b 0100 0361 6765  ng/String;...age
00000050: 0100 0149 0100 063c 696e 6974 3e01 0003  ...I...<init>...
00000060: 2829 5601 0004 436f 6465 0100 0f4c 696e  ()V...Code...Lin
00000070: 654e 756d 6265 7254 6162 6c65 0100 0369  eNumberTable...i
00000080: 6e63 0100 0428 4929 4a01 000d 5374 6163  nc...(I)J...Stac
00000090: 6b4d 6170 5461 626c 6507 001e 0700 2101  kMapTable.....!.
000000a0: 0008 3c63 6c69 6e69 743e 0100 0a53 6f75  ..<clinit>...Sou
000000b0: 7263 6546 696c 6501 000e 5465 7374 436c  rceFile...TestCl
000000c0: 6173 732e 6a61 7661 0c00 0e00 0f01 0005  ass.java........
000000d0: 6865 6c6c 6f0c 000a 000b 0c00 0c00 0d01  hello...........
000000e0: 0013 6a61 7661 2f6c 616e 672f 4578 6365  ..java/lang/Exce
000000f0: 7074 696f 6e01 0029 636f 6d2f 6864 6a6e  ption..)com/hdjn
00000100: 622f 636c 6173 7370 6172 7365 722f 7465  b/classparser/te
00000110: 7374 436c 6173 732f 5465 7374 436c 6173  stClass/TestClas
00000120: 7301 0010 6a61 7661 2f6c 616e 672f 4f62  s...java/lang/Ob
00000130: 6a65 6374 0100 136a 6176 612f 6c61 6e67  ject...java/lang
00000140: 2f54 6872 6f77 6162 6c65 0021 0008 0009  /Throwable.!....
```

在开始编写之前先解释一下上图中的u1, u2, u4是什么意思。它们表示的都是无符号数，后面的数字代表几个字节。比如u1就表示一个字节的无符号数，取值范围是[0, 255]（不懂的去复习一下计算机中如何表示数字的内容）。

提示：示例代码均是kotlin编写，我尽量不使用kotlin中比较特殊的语法，如果使用了会解释语法的意思。

首先在解析class文件之前，需要把class文件的字节流读入内存，而且由于class文件的组成部分有1,2,4,8字节大小，因此还需要读1,2,4,8字节的方法。新建**FileUtil**和**ByteReader**类，一个用于读取class文件字节流，一个用于读取指定大小的字节。

```kotlin
class FileUtil {
    companion object {
        fun readBytes(filePath: String): ByteArray {
            val classFile = FileUtils.getFile(filePath)
            return FileUtils.readFileToByteArray(classFile)
        }
    }
}
```

```kotlin
/**
 * kotlin和Java一样没有无符号数，因此需要先把Byte转换成无符号数UByte。转换后的UByte是10进制数，
 * 而class文件中的字节是16进制，因此还需要再转换一次。
*/
class ByteReader(private val filePath: String) {
    private var bytes: MutableList<Byte> = FileUtil.readBytes(filePath).toMutableList()

    fun readU1(): String = convertByteToHex(bytes.removeAt(0))

    fun readU2(): String {
        val u2 = bytes.slice(IntRange(0, 1))
        bytes = bytes.drop(2).toMutableList()
        return u2.map { convertByteToHex(it) }.reduce { sum, element ->
            "$sum$element"
        }
    }

    fun readU4(): String {
        val u4 = bytes.slice(IntRange(0, 3))
        bytes = bytes.drop(4).toMutableList()
        return u4.map { convertByteToHex(it) }.reduce { sum, element ->
            "$sum$element"
        }
    }

    fun readU8(): String {
        val u8 = bytes.slice(IntRange(0, 7))
        bytes = bytes.drop(8).toMutableList()
        return u8.map { convertByteToHex(it) }.reduce { sum, element ->
            "$sum$element"
        }
    }
}
```

**提示1：companion object等价与Java的静态方法**

**提示2："$variable"是字符串模板**

还需要一个进制转换的工具类**BaseUtil**

```kotlin
class BaseUtil {
    companion object {
        fun convertByteToHex(byte: Byte) = String.format("%02X", byte.toUByte().toInt())

        fun convertHexToInt(hex: String) = Integer.parseInt(hex, 16)

        fun convertHexToLong(hex: String) = java.lang.Long.parseLong(hex, 16)

        fun convertHexToString(hex: String): String =
        	String(DatatypeConverter.parseHexBinary(hex))
    }
}
```

基础工作都做完了，接下来就可以开始一步步解析class文件了。下篇见。