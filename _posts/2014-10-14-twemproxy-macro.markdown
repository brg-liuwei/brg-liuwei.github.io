---
layout: post
title: "提升C程序员的逼格：从twemproxy源码学习宏定义技巧"
date: 2014-10-14
categories: tech
---

[twemproxy][twemproxy_github_addr]是twitter开发的用于redis或memcache的proxy，其原理是对key计算hash，然后对节点数取模，就映射到了对应的redis/memcache节点上。当然如果要添加或者删除节点，需要手动去做数据迁移。

今天和前同事默罕默德•凯讨论twemproxy中的一段代码，颇有意思：

File: [src/nc_conf.c][nc_conf_c_addr]
{% highlight c linenos %}
#define DEFINE_ACTION(_hash, _name) string(#_name),
static struct string hash_strings[] = {
    HASH_CODEC( DEFINE_ACTION )
    null_string
};
#undef DEFINE_ACTION

#define DEFINE_ACTION(_hash, _name) hash_##_name,
static hash_t hash_algos[] = {
    HASH_CODEC( DEFINE_ACTION )
    NULL
};
#undef DEFINE_ACTION

{% endhighlight %}

其中HASH_CODEC的定义在：[src/hashkit/nc_hashkit.h][nc_hashkit_h_addr]
{% highlight c linenos %}
#define HASH_CODEC(ACTION)                      \
    ACTION( HASH_ONE_AT_A_TIME, one_at_a_time ) \
    ACTION( HASH_MD5,           md5           ) \
    ACTION( HASH_CRC16,         crc16         ) \
    ACTION( HASH_CRC32,         crc32         ) \
    ACTION( HASH_CRC32A,        crc32a        ) \
    ACTION( HASH_FNV1_64,       fnv1_64       ) \
    ACTION( HASH_FNV1A_64,      fnv1a_64      ) \
    ACTION( HASH_FNV1_32,       fnv1_32       ) \
    ACTION( HASH_FNV1A_32,      fnv1a_32      ) \
    ACTION( HASH_HSIEH,         hsieh         ) \
    ACTION( HASH_MURMUR,        murmur        ) \
    ACTION( HASH_JENKINS,       jenkins       ) \
{% endhighlight %}

刚看到这段宏定义的时候会有点点纳闷，因为在源代码中检索不到HASH_CODEC里面定义的ACTION。仔细一看，原来ACTION是HASH_CODEC传进来的参数，而在前面nc_conf.c中，先定义了“函数” DEFINE_ACTION，然后把DEFINE_ACTION作为“参数”传到HASH_CODEC中。其实是实现了一个类似把函数当成参数传递的功能，其实把hash_strings扩展开来，就相当于：
{% highlight c linenos %}
/* 这里的DEFINE_ACTION就是HASH_CODEC里面的ACTION，
   宏定义里面的#号表示把后面替换的符号转换成字符串
   （例如：string(#md5) ==> string("md5") ）
 */
#define DEFINE_ACTION(_hash, _name) string(#_name),
static struct string hash_strings[] = {
    string("one_at_a_time") \
    string("md5"), \
    string("crc16"), \
    string("crc32"), \
    string("crc32a"), \
    string("fnv1_64"), \
    string("fnv1a_64"), \
    string("fnv1_32"), \
    string("fnv1a_32"), \
    string("hsieh"), \
    string("murmur"), \
    string("jenkins"), \
    null_string
};
#undef DEFINE_ACTION

{% endhighlight %}

再去[src/nc_string.h][nc_string_h_addr]中看string定义，就一目了然了：
{% highlight c linenos %}
struct string {
    uint32_t len;   /* string length */
    uint8_t  *data; /* string data */
};

#define string(_str)   { sizeof(_str) - 1, (uint8_t *)(_str) }
#define null_string    { 0, NULL }
{% endhighlight %}

相同的道理，再来看这一段：
{% highlight c linenos %}
/* 宏定义中的##号表示把前后两个符号连接成一个符号（注意不是字符串），
   这里的_name是宏参数，注意hash_不是_hash，就仅仅是个符号而已，
   因此, ACTION(HASH_MD5, md5) 就直接展开成为 hash_md5
 */
#define DEFINE_ACTION(_hash, _name) hash_##_name,
static hash_t hash_algos[] = {
    HASH_CODEC( DEFINE_ACTION )
    NULL
};
#undef DEFINE_ACTION
{% endhighlight %}

在[src/nc_server.h][nc_server_h_addr]中，定义了hash_t:
{% highlight c linenos %}
typedef uint32_t (*hash_t)(const char *, size_t);
{% endhighlight %}

所以hash_algos是一个函数指针数组，这个宏定义展开就是：
{% highlight c linenos %}
static hash_t hash_algos[] = {
    hash_one_at_a_time \
    hash_md5, \
    hash_crc16, \
    hash_crc32, \
    hash_crc32a, \
    hash_fnv1_64, \
    hash_fnv1a_64, \
    hash_fnv1_32, \
    hash_fnv1a_32, \
    hash_hsieh, \
    hash_murmur, \
    hash_jenkins, \
    NULL
};
{% endhighlight %}

头文件中用先定义DEFINE_ACTION，把这个“函数”当成一个参数传入到另一个宏中，然后再undefine，就像是在用一个变量不停地赋值然后传入到另一个函数中去，只需要用一行代码去将DEFINE_ACTION定义成一个生成规则，再定义HASH_CODEC，通过这段让人眼花缭乱的代码，可以完成多个数组的赋值，不得不说这是一个C语言工程师提升逼格的必备技巧！


[twemproxy_github_addr]: https://github.com/twitter/twemproxy
[nc_conf_c_addr]: https://github.com/twitter/twemproxy/blob/master/src/nc_conf.c
[nc_hashkit_h_addr]: https://github.com/twitter/twemproxy/blob/master/src/hashkit/nc_hashkit.h
[nc_string_h_addr]: https://github.com/twitter/twemproxy/blob/master/src/nc_string.h
[nc_server_h_addr]: https://github.com/twitter/twemproxy/blob/master/src/nc_server.h
