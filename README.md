# Http from Tcp

[YouTube](https://www.youtube.com/watch?v=FknTw9bJsXM&t=1042s)

## HTTP streams

### Welcome

```sh
mkdir http-from-tcp
```

```go
package main

func main() {}
```

### Startup

```sh
go mod init github.com/mariolazzari/http-from-tcp
```

```go
import "fmt"

func main() {
	fmt.Println("I hope I get the job!")
}

```

### 8 bytes

```go
func main() {
	f, err := os.Open("messages.txt")
	if err != nil {
		log.Fatal("Error reading messages.txt")
	}
	defer f.Close()

	data := make([]byte, 8)

	for {
		n, err := f.Read(data)
		if n > 0 {
			fmt.Printf("read: %s\n", data[:n])
		}

		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

### New lines

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"os"
)

func main() {
	f, err := os.Open("messages.txt")
	if err != nil {
		log.Fatal("Error reading messages.txt")
	}
	defer f.Close()

	str := ""

	for {
		data := make([]byte, 8)
		n, err := f.Read(data)
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatal(err)
		}

		data = data[:n]
		if i := bytes.IndexByte(data, '\n'); i != -1 {
			str += string(data[:i])
			data = data[i+1:]
			fmt.Printf("read: %s\n", str)
			str = ""
		}

		str += string(data)
	}

	if len(str) != 0 {
		fmt.Printf("read: %s\n", str)
	}
}
```

### Channel refactor

[State machine](https://developer.mozilla.org/en-US/docs/Glossary/State_machine)

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"os"
)

func main() {
	f, err := os.Open("messages.txt")
	if err != nil {
		log.Fatal("Error reading messages.txt")
	}

	for line := range getLinesChannel(f) {
		fmt.Printf("read: %s\n", line)
	}
}

func getLinesChannel(f io.ReadCloser) <-chan string {
	out := make(chan string)
	str := ""

	go func() {
		defer close(out)
		defer f.Close()

		for {
			data := make([]byte, 8)
			n, err := f.Read(data)
			if err == io.EOF {
				break
			}
			if err != nil {
				log.Fatal(err)
			}

			data = data[:n]
			if i := bytes.IndexByte(data, '\n'); i != -1 {
				str += string(data[:i])
				data = data[i+1:]
				fmt.Printf("read: %s\n", str)
				str = ""
			}

			str += string(data)
		}
	}()

	return out
}
```

## TCP

[Tcp](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
[Http3](https://www.cloudflare.com/learning/performance/what-is-http3/)
[net](https://pkg.go.dev/net#TCPConn)
[Accept](https://pkg.go.dev/net#Listener.Accept)
[tee](<https://en.wikipedia.org/wiki/Tee_(command)>)

### Tcp

```sh
go run . | tee /tmp/tcp.txt
nc localhost 42069
```

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"net"
)

func main() {
	listener, err := net.Listen("tcp", ":42069")
	if err != nil {
		log.Fatal("error", "error", err)
	}
	defer listener.Close()

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Fatal("error", "error", err)
		}

		for line := range getLinesChannel(conn) {
			fmt.Printf("read: %s\n", line)
		}
	}
}

func getLinesChannel(f io.ReadCloser) <-chan string {
	out := make(chan string)
	str := ""

	go func() {
		defer close(out)
		defer f.Close()

		for {
			data := make([]byte, 8)
			n, err := f.Read(data)
			if err == io.EOF {
				break
			}
			if err != nil {
				log.Fatal(err)
			}

			data = data[:n]
			if i := bytes.IndexByte(data, '\n'); i != -1 {
				str += string(data[:i])
				data = data[i+1:]
				fmt.Printf("read: %s\n", str)
				str = ""
			}

			str += string(data)
		}
	}()

	return out
}
```

### Tcp vs Udp

[Udp](https://en.wikipedia.org/wiki/User_Datagram_Protocol)
[ResolveUDPAddr](https://pkg.go.dev/net#ResolveUDPAddr)

```sh
go run ./cmd/tcplistener
nc -v localhost 42069
go run ./cmd/tcplistener | tee /tmp/tcplistener.txt
```

```go
package main

import (
	"bufio"
	"log"
	"net"
	"os"
)

func main() {
	addr, err := net.ResolveUDPAddr("udp", "localhost:42069")
	if err != nil {
		log.Fatal(err)
	}

	conn, err := net.DialUDP("udp", nil, addr)
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	reader := bufio.NewReader(os.Stdin)

	for {
		print("> ")

		line, err := reader.ReadString('\n')
		if err != nil {
			log.Println(err)
			continue
		}

		_, err = conn.Write([]byte(line))
		if err != nil {
			log.Println(err)
		}
	}
}
```
