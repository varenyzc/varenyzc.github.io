---
title: Kotlin学习笔记
date:  2020-10-23 17:08:49 +0800
category: develop 
tags: Kotlin
excerpt: 学习kotlin期间的一些笔记
---

## 一、拓展函数
### 1、run
run函数接收一个函数参数并作为该函数的返回值作为run函数的返回值
```kotlin
fun main(){
      var nickName = "varenyzc"
        nickName = nickName.run {
                  if(isNotEmpty()){
                        this
                  } else {
                        ""
                  }
        }
        println(nickName) //varenyzc
}
```

### 2、with
with 函数并不是扩展函数，不过由于作用相近，此处就一起介绍了。with 函数用于对同一个对象执行多次操作而不需要反复把对象的名称写出来。

e.g.，为了构建一个包含指定内容的字符串，需要先后如下调用
```kotlin
fun main() {
    val result = StringBuilder()
    result.append("varenyzc")
    result.append("\n")
    for (letter in 'A'..'Z') {
        result.append(letter)
    }
    println(result.toString())
 }
```
改为通过 with 函数来构建的话会代码会简洁许多
```kotlin
val result = with(StringBuilder()) {
        append("varenyzc")
        append("\n")
        for (letter in 'A'..'Z') {
            append(letter)
        }
        toString()
    }
    println(result)
```
with 函数是一个接受两个参数的函数，在这个例子中就是一个 StringBuilder 和一个 Lambda 表达式，这里利用了把 Lambda 表达式放在括号外的约定。

### 3、apply
apply 函数被声明为类型 T 的扩展函数，它的接收者是作为实参的 Lambda 的接受者，最终函数返回 this 即对象本身。
所以apply 函数和 with 函数的唯一区别在于：apply 函数始终会返回作为实参传递给它的对象。
```kotlin
val result = StringBuilder().apply {
        append("varenyzc")
        append("\n")
        for (letter in 'A'..'Z') {
            append(letter)
        }
        toString()
    }
    println(result)
    println(result.javaClass) //class java.lang.StringBuilder
```

### 4、also
also 函数接收一个函数类型的参数，该参数又以接收者本身作为参数，最终返回接收者对象本身
```kotlin
fun main() {
    val nickName = "varenyzc"
    val also = nickName.also {
        it.length
    }
    println(also) //varenyzc
}
```

### 5、let
let 函数接收一个函数类型的参数，该参数又以接收者本身作为参数，最终返回函数的求值结果
```kotlin
fun main() {
    val nickName = "varenyzc"
    val also = nickName.let {
        it.length
    }
    println(also) //8
}
```

### 6、takeIf
takeIf 接收一个返回值类型为 bool 的函数，当该参数返回值为 true 时返回接受者对象本身，否则返回 null
```kotlin
fun main() {
    println(check("varenyzc")) //8
    println(check(null)) //0
}

fun check(name: String?): Int {
    return name.takeIf { !it.isNullOrBlank() }?.length ?: 0
}
```

### 7、takeUnless
takeUnless 的判断条件与 takeIf 相反，这里不再赘述




