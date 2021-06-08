## Golang for循环

Go中循环一般有两种写法 for与range，其实range是类似于语法糖，编译器在编译阶段会将range转换成普通的for循环

### Range循环的乱象

- 循环永动机
```go
package main

import "fmt"

func foo() {
	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}

	fmt.Println(arr)
}

// 输出结果 1 2 3 1 2 3
```
上述结果表明我们在遍历切片时追加元素并不会影响原有的循环逻辑，原因在于go在编译阶段将原切片进行复制到变量ha，
而且在转化后的普通for循环中是预先取了len()长度，所以循环次数就是固定的。类似于下面的伪代码

```go
package main

import "fmt"

func foo() {
	arr := []int{1, 2, 3}
	ha := arr
	size := len(arr)
	for i := 0; i < size; i++ {
		arr = append(arr, ha[i])
	}

	fmt.Println(arr)
}
```


- 神奇的指针

```go
package main

import "fmt"

func foo() {
	arr := []int{1, 2, 3}
	var newArr []*int

	for _, v := range arr {
		newArr = append(newArr, &v)
	}

	for _, v := range newArr {
		fmt.Println(*v)
	}
}

// 输出结果 3 3 3
```
因为在range遍历切片时，Go语言将数组中的变量拷贝给了v，且在整个循环过程中只存在这一个v，
在循环过程中v被不断的重新赋值，所以会产生该现象