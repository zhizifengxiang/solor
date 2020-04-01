# 1 Use pthread

pthread 库向用户提供了各种数据类型和函数，使用户可以创建线程，并在线程中运行不同的函数。下面简单介绍集中数据类型和函数。

|数据类型、函数|
|---|
|pthread_t|
|int pthread_create(pthread_t *threadId, const pthread_attr_t *attr, void *(*func)(void *), void *arg)|
|int pthread_exit(void *valuePtr)|
|int pthread_detach(pthread_t threadId)|
|int pthread_join(pthread_t threadId, void **valuePtr)|
|pthread_t pthread_self(void)|
|int sched_yield(void)|

在上表中，pthread_t类型唯一标识一个

