######ngx_aio_read.c
`struct ngx_event_s`里的`complete`，`ready`，`active`分别在什么时候赋值了以及什么时候用到了？
active表示aio_read正在进行中，ready表示一次read完成了，可以进行下一次aio_read了，complete=0表示还没有启动aio_read，可以启动aio_read。

######ngx_errno.c

    p = malloc(len);
    if (p == NULL) {
        goto failed;
    }
    
但是failed并没有free(p)，是因为不成功就退出程序，所以不管么？