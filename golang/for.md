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
