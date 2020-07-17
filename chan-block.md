~~~golang
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

或者使用 `context`

~~~golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"context"
	_ "net/http/pprof"
	"time"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithCancel(r.Context())
		defer cancel()

		ch := make(chan bool)
		//fmt.Println("xxx")

		go func() {
			time.Sleep(1000 * time.Millisecond)
			select {
				case <-ctx.Done():
					fmt.Println("ctx done")
					return
				case ch <- true:
					fmt.Println("done")
			}
		}()


		timeout := time.After(1800* time.Millisecond)
		select {
		case x := <-ch:
            fmt.Println("result", x)
		case <-timeout:
		}
		fmt.Fprint(w, "hello world")
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
~~~
