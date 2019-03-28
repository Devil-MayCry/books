

### pprof

Golang标准库里，有一个叫做

`pprof`

的包，通过这个包，我们可以profiling任意的程序，两个函数调用即可。

```
import (

     "runtime/pprof"
）

func main() {
  ....
  f, err := os.Create("cpu-profile.prof")
  if err != nil {
    log.Fatal(err)
  }
  pprof.StartCPUProfile(f)
  ... // this is program you want to profile
  pprof.StopCPUProfile()
}
```

然后运行

`go tool pprof cpu-profile.prof`

输入

`web`

即可达到cpu分析图



### 火焰图

用得到的pprof即可生成火焰图（需要先安装go-torch）



` go-torch --binaryname=./pprof_runtime --binaryinput=cpu-profile.prof`



**参考**

http://www.hatlonely.com/2018/01/29/golang-pprof-%E6%80%A7%E8%83%BD%E5%88%86%E6%9E%90%E5%B7%A5%E5%85%B7/index.html



http://cjting.me/golang/use-pprof-to-optimize-go/

