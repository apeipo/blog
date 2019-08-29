---
title: 使用gdb查看php进程正在执行的代码 
date: 2016-03-28 23:38:42
tags: [PHP, GDB]
category: PHP
---
## 参考链接
[找出php中可能有问题的代码行](http://www.searchtb.com/2014/04/%E5%BD%93cpu%E9%A3%99%E5%8D%87%E6%97%B6%EF%BC%8C%E6%89%BE%E5%87%BAphp%E4%B8%AD%E5%8F%AF%E8%83%BD%E6%9C%89%E9%97%AE%E9%A2%98%E7%9A%84%E4%BB%A3%E7%A0%81%E8%A1%8C.html)
[使用GDB调试PHP代码，解决PHP代码死循环](http://rango.swoole.com/archives/325)
## 运行时变量
当某个php进程僵死或者长时间占用CPU比较高时，我们需要知道该进程正在执行的代码以确定问题原因。
php在解释执行过程中，zend引擎用`executor_globals`变量保存了执行过程中的各种数据，包括执行函数、文件、代码行等。
`executor_globals` 中的`active_op_array` 和 `current_execute_data`变量保存了当前的执行信息。
相关定义如下，可以看到`active_op_array`中有执行时的`function_name`和`filename`，`current_execute_data`中有执行行号。

```c
//active_op_array
struct _zend_op_array {
    /* Common elements */
    zend_uchar type;
    zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
    uint32_t fn_flags;
    zend_string *function_name;//function name
    zend_class_entry *scope;
    zend_function *prototype;
    uint32_t num_args;
    uint32_t required_num_args;
    zend_arg_info *arg_info;
    /* END of common elements */

    uint32_t *refcount;

    uint32_t this_var;

    uint32_t last;
    zend_op *opcodes;

    int last_var;
    uint32_t T;
    zend_string **vars;

    int last_brk_cont;
    int last_try_catch;
    zend_brk_cont_element *brk_cont_array;
    zend_try_catch_element *try_catch_array;

    /* static variables support */
    HashTable *static_variables;

    zend_string *filename;//execute file
    uint32_t line_start;
    uint32_t line_end;
    zend_string *doc_comment;
    uint32_t early_binding; /* the linked list of delayed declarations */

    int last_literal;
    zval *literals;

    int  cache_size;
    void **run_time_cache;

    void *reserved[ZEND_MAX_RESERVED_RESOURCES];
};

//current_execute_data
struct _zend_execute_data {
    const zend_op       *opline;           /* executed opline                */
    zend_execute_data   *call;             /* current call                   */
    zval                *return_value;
    zend_function       *func;             /* executed funcrion              */
    zval                 This;             /* this + call_info + num_args    */
    zend_class_entry    *called_scope;
    zend_execute_data   *prev_execute_data;
    zend_array          *symbol_table;
#if ZEND_EX_USE_RUN_TIME_CACHE
    void               **run_time_cache;   /* cache op_array->run_time_cache */
#endif
#if ZEND_EX_USE_LITERALS
    zval                *literals;         /* cache op_array->literals       */
#endif
};

//opline
struct _zend_op {
    opcode_handler_t handler;
    znode result;
    znode op1;
    znode op2;
    ulong extended_value;
    uint lineno;
    zend_uchar opcode;
};

```
## 用gdb查看变量
创建一个php文件，用cli方式执行，逻辑就是简单的sleep。

```php
function fck(){
   while(true){
       echo "hehe";
       sleep(1);
   }
}
fck();
```
执行过程用gdb的-p参数调试进程，**这里得注意，如果php在编译时没有开启debug选项，看到的调试信息中无法获取到执行中的参数**
>执行gdb后，死循环的进程会变成T的状态，表示正在Trace。这个是独占的，所以不能再使用strace/gdb或者其他ptrace工具对此进程进行调试。另外此进程会中断执行

执行如下图，可以看到，该进程正在执行的是test.php的fck方法
![](https://longlog-1300108443.cos.ap-beijing.myqcloud.com/before2019/2016-03-28-14591791052029.jpg)

另外，php的源码根目录有提供.gdbinit文件，是一个gdb的脚本，封装了上诉过程，在gdb中source该文件后使用zbacktrace可以直接看到当前执行函数、文件名和行数。

```sh
source php_src/.gdbinit
zbacktrace
```



