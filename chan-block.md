~~~
package main

import (
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof"
	"time"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		ch := make(chan bool, 1) // 最重要的在这里，参考：https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html
		go func() {
			time.Sleep(100 * time.Millisecond)
			ch <- true
			log.Println("i exit")
		}()

		timeout := time.After(10 * time.Millisecond)
		select {
		case <-ch:
		case <-timeout:
		}
		fmt.Fprint(w, "hello world")
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
~~~

如果用`make(chan bool)`而非`make(chan bool, 1)`的话，因timeout造成`ch`关闭了无法写入，就会造成写入阻塞产生goroutine leak
