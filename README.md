# OSLab3
Buaa Os lab3
##env_init##
```C
void
env_init(void)
{
       int i;

/*precondition: envs pointer has been initialized at mips_vm_init, called by mips_init*/
       /*1. initial env_free_list*/
       LIST_INIT(&env_free_list);
       
       /*2. Travel the elements in 'envs', initial every element(mainly initial its status, mark it as free)  
       and inserts them into the env_free_list. attention :Insert in reverse order */
       
       for(i=NENV-1;i>=0;i--){
             envs[i].env_status = ENV_FREE;
             LIST_INSERT_HEAD(&env_free_list,envs+i,env_link);
       }

}
```
以上是env_init的实现。其实这个函数没什么太多好说的，就是初始化`env_free_list`，然后按逆序插入`envs[i]`。  
这里唯一值得并需要引起警惕的是逆序，因为我们使用的是`LIST_INSERT_HEAD`这个宏，任何一个对齐有所了解的人应该都知道，这个宏每次都会将一个结点插入，变成链表的第一个可用结点，我们在取用的时候是使用`LIST_FIRST`宏来取的。  
可能会有同学问为什么`NENV`是envs的长度，这个实际上在`./mm/pmap.c`里面的`mips_vm_init`里可以找到我们的证据，证明`envs`数组确实给它分配了`NENV`个结构体的空间，所以它也就有`NENV`个元素了。
