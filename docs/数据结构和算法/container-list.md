## 问题

语言自带的双向链表

## 解决方案


来自[The list container in Go](https://golangdocs.com/list-container-in-go)

`container/list`是golang自带的双向链表数据类型，`import` 之后可以直接使用。

本身定义了两个`struct`,一个是链表的数据项，一个是链表本身。具体[文档](https://pkg.go.dev/container/list)


### 初始化一个链表

初始化一个空链表。当链表未存储内容的时候，链表的头尾将会是`nil`

```go
l := list.New()    // 初始化一个空链表
fmt.Println(l)      // &{{0x43e280 0x43e280 <nil> <nil>} 0}
```

### 链表头尾数据

`front`和`back` 两个函数用于获取链表头尾数据. 下面是但链表为空的时候，头尾皆为`nil`.

```go
fmt.Println(l.Front())      // <nil>
fmt.Println(l.Back())       // <nil>
```

### 向链表中添加数据项

有多种方式往链表中添加数据项，下面是两个样例

#### 1. 头部添加数据项

`PushFront`函数接收一个数据项将其添加到列表的头部.

```go
l.PushFront(10)
fmt.Println(l.Front())       // &{0x43e280 0x43e280 0x43e280 10}
```

整个链表可以整体通过`PushFrontList`函数被push到另外一个链表的头部，需要注意的事被插入的链表需要先正常初始化

```go
l.PushFrontList(l2)    // where l2 is another list

#### 2. 尾部添加数据项

`PushBack`函数接收一个数据项并将其插入到链表尾部

```go
package main

import (
	"fmt"
	"container/list"
)

func main() {
	l := list.New()
	l.PushBack(10)
	l.PushBack(12)
	l.PushBack(14)
	for e := l.Front(); e != nil; e = e.Next() {
		fmt.Println(e)
	}
	
	// &{0x43e2c0 0x43e280 0x43e280 10}
	// &{0x43e2e0 0x43e2a0 0x43e280 12}
	// &{0x43e280 0x43e2c0 0x43e280 14}
}
```

`PushBackList`和`PushFrontList`类似，只不过讲一个链表完整地插入到另一个链表的尾部

```go
l.PushBackList(l2)  // l2 is a list
```
### 从链表中移除数据项

The remove function simply removes the element passed into it from the list. Here it is in action.

```go
package main

import (
	"fmt"
	"container/list"
)

func main() {
	l := list.New()

        // 存储引用
	v := l.PushBack(10)
	fmt.Println(l.Front())      // &{0x43e280 0x43e280 0x43e280 10}
       
        // 移除引用
	l.Remove(v)
	fmt.Println(l.Front())      // <nil>
}
```


