# Redis

## 1. SDS

```c
struct __attribute ((__packed__)) sdshdr8 {
	uint8_t len;    /* used */
	uint8_t alloc;  /* excluding the header and null terminator */
	unsigned char flags;
	char buf[];
};

struct __attribute ((__packed__)) sdshdr16 {...};
struct __attribute ((__packed__)) sdshdr32 {...};
struct __attribute ((__packed__)) sdshdr64 {...};
```

### 1.1. 说明SDS和C字符串的不同之处，解释为什么Redis要使用SDS而不是C字符串？

                 C字符串                     |                    SDS 
---------------------------------------------|--------------------------------------------
获取字符串长度的复杂度为O(N)                 |  获取字符串长度的复杂度为O(1)
API是不安全的，可能会造成缓冲区溢出          |  API是安全的，不会造成缓冲区溢出
修改字符串长度N次必然需要执行N次内存重分配   |  修改字符串长度N次最多需要执行N次内存重分配
只能保存文本数据                             |  可以保存文本或者二进制数据
可以使用所有<string.h>库中的函数             |  可以使用部分<string.h>库中的函数
