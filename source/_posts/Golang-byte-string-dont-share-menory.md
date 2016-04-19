title: Golang中[]Byte与string类转换时内存并不共享
date: 2016-04-19 20:51
tags:
- Golang
- 技术
categories: 技术
---

在Golang的基础中我们知道在用append的时候，或者Slice底层数组小于需求容量的时候，Go会自动重新分配内存，随之带来的有一次值拷贝。在数据量小或函数调用不频繁的时候对性能的损失并不明显，但是如果在高频率调用或者数据量大的时候，性能损失不能忽略不计。

那么那些情况会触发值拷贝呢？我们来逐一测试：

1. []byte截取：

    ```Golang
    pckage main

    import "fmt"

    func main() {
        str := []byte("this is a fucking string")
        str2 := str[0:5]
        str3 := str2[0:2]
        fmt.Printf("%p\n", str)
        fmt.Printf("%p\n", str2)
        fmt.Printf("%p\n", str3)
    }
    ```

    输出：  

    > 0xc82000e4e0  
    > 0xc82000e4e0  
    > 0xc82000e4e0  

    分析：  
    对[]byte进行了两次切片操作，首地址不变。这也说明了这三个Slice使用了同一个底层数组，并不发生值拷贝。

2. string与[]byte互相转换:

    ```Golang
    package main

    import "fmt"

    func main() {
        str := []byte("this is a fucking string")
        str2 := string(str)
        str3 := []byte(str2)
        fmt.Printf("%p\n", str)
        fmt.Printf("%p\n", &str2)
        fmt.Printf("%p\n", str3)
    }

    ```

    输出：  

    > 0xc82000e4e0  
    > 0xc82000a330  
    > 0xc82000e520  

    分析：  
    无论从[]byte到string还是string到[]byte，他们的指针地址均不同。说明在类型转换的时候，发生了值拷贝，而[]byte与string并不共享内存。

    查阅资料，发现老外有一篇[文章]也证明了这个观点：

    从Golang Runtime源代码来看，[]byte与string的转换分别调用了以下两个函数：
    - runtime.stringtoslicebyte()
    - runtime.slicebytetostring()

    ```Golang
    func slicebytetostring(b Slice) (s String) {
        void *pc;

        if(raceenabled) {
            pc = runtime·getcallerpc(&b);
            runtime·racereadrangepc(b.array, b.len, pc, runtime·slicebytetostring);
        }
            s = gostringsize(b.len);
            runtime·memmove(s.str, b.array, s.len);
    }

    func stringtoslicebyte(s String) (b Slice) {
        b.array = runtime·mallocgc(s.len, 0, FlagNoScan|FlagNoZero);
        b.len = s.len;
        b.cap = s.len;
        runtime·memmove(b.array, s.str, s.len);
    }
    ```

    可以看到其中menmove这些内存移动操作。

[文章]: http://golang-examples.tumblr.com/post/86403044869/conversion-between-byte-and-string-dont-share
