# Golang中的int类型和int32有啥区别

```go
package main

import (
        "fmt"
        "unsafe"
)

func main() {
        var i1 int = 1
        var i2 int8 = 
        var i3 int16 = 3
        var i4 int32 = 4
        var i5 int64 = 5
        fmt.Println(unsafe.Sizeof(i1))
        fmt.Println(unsafe.Sizeof(i2))
        fmt.Println(unsafe.Sizeof(i3))
        fmt.Println(unsafe.Sizeof(i4))
        fmt.Println(unsafe.Sizeof(i5))
}
```

```go
[root@localhost tmp]# go run main.go 
8
1
2
4
8
```

我的机器是64位机器，即一次cpu取数可以去到8个字节

可以看到int与int64大小是一致的，即等于CPU的位数。这么做的原因：

在不在乎int大小的场景下（数字范围不会特别大），使用int对于cpu兼容性是最好的。

> 
