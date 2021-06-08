## Golang defer

### 先贴一段有意思的代码
```go
package main

func createPost(db *gorm.DB) (err error) {
	tx := db.Begin()
	defer tx.Rollback()

	if err := tx.Create(&Post{Author: "fizz"}).Error; err != nil {
		return err
	}

	return tx.Commit().Error
}
```
当我们使用事务时，可以使用上面的写法。上面的代码一定会触发rollback。但是当commit成功后，rollback也不会对已经提交的事务产生影响

defer经常被使用于关闭链接，解锁等场景，使我们的编码变得非常的清爽。下面有一些坑点需要避免

### 现象
- defer的执行顺序，defer是压栈逻辑所以是倒叙执行，在panic之前声明的defer会被执行，之后的则不会

```go
package main

import "fmt"

func foo() {
	defer fmt.Println(1)
	defer fmt.Println(2)
	defer fmt.Println(3)
	panic("stop")
	defer fmt.Println(4)
}

// 执行结果 1 2 3 panic
```

- 预计算参数

```go
package main

import (
	"fmt"
	"time"
)

func foo() {
	startTime := time.Now()
	defer fmt.Println(time.Since(startTime))
	
	time.Sleep(time.Second)
}

// 执行结果 0s
```
defer在执行是会立刻拷贝函数中引用的外部参数并进行计算，所以time.Since()被立刻执行了。只是打印在之后执行而已。改进后的代码如下
```go
package main

import (
	"fmt"
	"time"
)

func foo() {
	startTime := time.Now()
	defer func() {
		fmt.Println(time.Since(startTime))
    }()
	
	time.Sleep(time.Second)
}
// 执行结果 1s
```
在改进后的代码中被拷贝的是函数指针fmt.Println。time.Since被延迟计算

### defer的优化
defer开放编码，在栈上执行defer，当全部满足下面条件时触发
- 函数中的defer数量少于8个
- 函数中的defer不在循环中
- 函数的return语句与defer语句的乘积小于15