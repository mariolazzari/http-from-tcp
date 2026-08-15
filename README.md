# Http from Tcp

## HTTP streams

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
	"fmt"
	"io"
	"log"
	"os"
	"strings"
)

func main() {
	f, err := os.Open("messages.txt")
	if err != nil {
		log.Fatal("Error reading messages.txt")
	}
	defer f.Close()

	data := make([]byte, 8)
	curLine := ""

	for {
		n, err := f.Read(data)
		if n > 0 {
			tokens := strings.Split(string(data[:n]), "\n")
			curLine += tokens[0]

			if len(tokens) == 2 {
				fmt.Printf("read: %s\n", curLine)
				curLine = tokens[1]
			}
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

### Channel refactor

```go
package main

import (
	"fmt"
	"io"
	"log"
	"os"
	"strings"
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
	ch := make(chan string)
	data := make([]byte, 8)
	curLine := ""

	go func() {
		for {
			n, err := f.Read(data)
			if n > 0 {
				tokens := strings.Split(string(data[:n]), "\n")
				curLine += tokens[0]

				if len(tokens) == 2 {
					ch <- curLine
					curLine = tokens[1]
				}
			}

			if err == io.EOF {
				close(ch)
				f.Close()
				break
			}

			if err != nil {
				log.Fatal(err)
			}
		}
	}()

	return ch
}
```
