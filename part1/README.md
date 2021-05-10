
# file

### Makefile
```
CC = gcc
CFLAGS = -W -Wall
TARGET = hello
OBJECTS = hello.o

all : $(TARGET)

$(TARGET) : $(OBJECTS)
	$(CC) $(CFLAGS) -o $@ $^

clean : 
	rm  *.o $(TARGET)
```
### hello compile
```
#include<stdio.h>
int main (int argc, char *agrv[]){
    printf("hello, world\n");
    return 0;
}
```

```
$ ls -l
-rwxr-xr-x 1 good dev 8304  5월 10 22:32 hello
-rw-r--r-- 1 good dev   98  5월 10 22:18 hello.c
-rw-r--r-- 1 good dev 1560  5월 10 22:32 hello.o
```
### ELF type
```
$ file hello
hello: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=4c2f1013baa212fae367413fac48e0eda97e6848, not stripped
```

### dynamic link library
```
#include<stdio.h>
int main (int argc, char *agrv[]){
    printf("hello, world\n");
    return 0;
}
```