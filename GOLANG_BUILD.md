# Cross-compiling Go binaries for the RH25v2

## Overview

The CPU is MIPS32LE:

```
# cat /proc/cpuinfo
system type             : halley2_plus_v10
machine                 : Unknown
processor               : 0
cpu model               : Ingenic Xburst V4.15  FPU V0.0
BogoMIPS                : 1001.88
wait instruction        : yes
microsecond timers      : no
tlb_entries             : 32
extra interrupt vector  : yes
hardware watchpoint     : yes, count: 1, address/irw mask: [0x0fff]
isa                     : mips32r1
ASEs implemented        :
shadow register sets    : 1
kscratch registers      : 0
core                    : 0
VCED exceptions         : not available
VCEI exceptions         : not available

Hardware                : halley2_plus
Serial                  : 00000000 00000000 00000000 00000000
```

Go has first class support for cross-compiling and supports MIPS as a target out of the box.

## Building Go code

You can build a simple hello world like this:

`go.mod`:

```
module helloworld

go 1.24.6
```

`main.go`

```
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

Then run:

```
env GOOS=linux GOARCH=mipsle go build
```

And copy the resulting binary to the optic using FTP. Set the executable bit and you're done:

```
# chmod +x /storage/helloworld
# /storage/helloworld
Hello, World!
```
