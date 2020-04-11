---
layout: post
title: 自制Java Class文件解析器之开始解析
date: 2019-04-11 17:03
author: zsh
catalog: true
tags:
    - Java
    - JVM
---

上文已经准备好了读字节码的工具ByteReader，同时也知道了class文件的组成部分，下面就可以开始一步步解析了。
首先创建一个类`ClassInfo`表示类文件
```kotlin
class ClassInfo(private val magicNumberInfo: MagicNumberInfo,
                private val minorVersionInfo: MinorVersionInfo,
                private val majorVersionInfo: MajorVersionInfo,
                private val constantPool: ConstantPool,
                private val accessFlagsInfo: AccessFlagsInfo,
                private val classExtensionInfo: ClassExtensionInfo,
                private val fieldsCount: Int,
                private val fields: List<FieldInfo>,
                private val methodsCount: Int,
                private val methods: List<MethodInfo>,
                private val attributesCount: Int,
                private val attributes: List<AttributeInfo>) {

    override fun toString(): String {
        return ""
    }
}
```
构造方法里的便是class文件里的各个部分。还需要一个解析器`Parser`，用来解析class文件中的各个部分。
```kotlin
class Parser(private val byteReader: ByteReader) {
    fun parseMagicNumber(): MagicNumberInfo = MagicNumberInfo(byteReader.readU4())

    fun parseMinorVersion(): MinorVersionInfo =
        MinorVersionInfo(convertHexToInt(byteReader.readU2()))

    fun parseMajorVersion(): MajorVersionInfo =
        MajorVersionInfo(convertHexToInt(byteReader.readU2()))
}
```
前面3个部分，魔数，次版本号，主版本号很简单，只要读取相应长度的字节即可，然后是重头戏常量池，根据上文可知，常量池先是一个2字节的数，表示常量池
中常量个数N，后面紧跟着N个常量。因此还需要一个枚举类，用来表示常量的类型。下面是截取了部分类型
```$kotlin
enum class Tag(val flag: Int,
               val tagName: String) {
    CONSTANT_UTF8_INFO(1, "Utf8"),
    CONSTANT_INTEGER_INFO(3, "Integer"),
}
```
然后便可以开始读取
```kotlin
fun parseConstPool(): ConstantPool {
    var index = 1
    val constantPoolCount = convertHexToInt(byteReader.readU2())
    val constants = arrayOfNulls<ConstantInfo?>(constantPoolCount)
    while (index < constantPoolCount) {
        val flag = convertHexToInt(byteReader.readU1())
        when (flag) {
            Tag.CONSTANT_CLASS_INFO.flag -> {
                constants[index] = parseConstantClassInfo()
                index++
            }
        }
    }
}
private fun parseConstantClassInfo(): ConstantClassInfo {
    val nameIndex = convertHexToInt(byteReader.readU2())
    return ConstantClassInfo(Tag.CONSTANT_CLASS_INFO, nameIndex)
}
```
这里我们使用数组临时保存解析出来的常量，同时下标从1开始。每解析完一项，下标+1。这里要注意一点，Double和Long类型的常量会占据常量池中的2格空间，
因此在解析Double和Long类型的常量时，index要+2。解析单个常量就是按照常量的格式读取字节，这里就不多讲了。
```kotlin
ConstantMethodRefInfo(tag=CONSTANT_METHOD_REF_INFO, classInfoIndex=9, nameAndTypeInfoIndex=26)
ConstantStringInfo(tag=CONSTANT_STRING_INFO, index=27)
ConstantFieldRefInfo(tag=CONSTANT_FIELD_REF_INFO, classInfoIndex=8, nameAndTypeInfoIndex=28)
ConstantFieldRefInfo(tag=CONSTANT_FIELD_REF_INFO, classInfoIndex=8, nameAndTypeInfoIndex=29)
Long			5
ConstantClassInfo(tag=CONSTANT_CLASS_INFO, nameIndex=30)
ConstantClassInfo(tag=CONSTANT_CLASS_INFO, nameIndex=31)
ConstantClassInfo(tag=CONSTANT_CLASS_INFO, nameIndex=32)
Utf8			name
Utf8			Ljava/lang/String;
Utf8			age
```
解析出来的结果

常量池解析完之后就是类的访问标识符，类自身信息，父类信息以及实现的接口，循规蹈矩读取即可。后续的字段，方法以及属性下篇继续解析。