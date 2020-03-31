# 我的golang代码规范

俗话说的好，细节决定成败。Coding过程中的规范就是这种可以决定成败的细节，好的规范可以是代码的可读性和可维护性都得到极大的增强，下面是我个人在工作中对自己的要求和规范，主要遵循一致性的原则。

## 1、包（package）

### 1.1 包的命名

- 全部小写。没有大写或下划线。
- 简短而简洁。请记住，在每个使用的地方都完整标识了该名称。
- 不用复数。例如`net/url`，而不是`net/urls`。
- 使用语义比较强的命名，而不要使用信息量不足，看不懂的命名。

### 1.2 包的 import 格式

对包的导入进行分组，标准库和第三方库

如：

```go
import (
  "fmt"
  "os"

  "github.com/gobwas/ws"
  "golang.org/x/sync/errgroup"
)
```

## 2、函数

### 1.1 函数的命名

- 语义明确（看到名字就能知道函数的作用和功能）.
- 驼峰式命名方式.(导出型的首字母必须大写).
- 尽量使用动词或者动词短语.

### 1.2 函数的分组与顺序

- 导出的函数先出现在文件中

- 同一文件中的函数按照接受者分组

  ```go
  type something struct{ ... }
  
  func Run()error{
    ...
    return nil
  }
  func newSomething() *something {
      return &something{}
  }
  
  func (s *something) Cost() {
    return calcCost(s.weights)
  }
  
  func (s *something) Stop() {...}
  
  func calcCost(n []int) int {...}
  ```

### 1.3 函数内部的规范

- 减少嵌套

  ```go
  for _, v := range data {
    if v.F1 != 1 {
      log.Printf("Invalid v: %v", v)
      continue
    }
  
    v = process(v)
    if err := v.Call(); err != nil {
      return err
    }
    v.Send()
  }
  ```

> 优先处理了错误和特殊情况，尽早的返回来减少嵌套

- 函数内部变量设置明确的值，尽量使用短变量声明形式 (`:=`).

```go
s  := "hello,world"
```

- 函数内部尽量缩小变量的作用域

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}//err变量只在这行代码中有效
```

- 避免函数的参数语意不明

（未完待续...）