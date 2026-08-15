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
