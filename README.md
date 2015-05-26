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
##env_steup_vm##
```C
实验指导书试行模式?
static int
env_setup_vm(struct Env *e)
{
       // Hint:

       int i, r;
       struct Page *p = NULL;

       Pde *pgdir;
       if ((r = page_alloc(&p)) < 0)
       {
               panic("env_setup_vm - page_alloc error\n");
                       return r;
       }
       p->pp_ref++;
       e->env_pgdir = (void *)page2kva(p);
       e->env_cr3 = page2pa(p);
       static_assert(UTOP % PDMAP == 0);
       for (i = PDX(UTOP); i <= PDX(~0); i++)
       
         (Hint1:这里为什么是UTOP,而不是ULIM,UTOP--->ULIM这段区域是干什么用的?)
         
         e->env_pgdir[i] = boot_pgdir[i];
       e->env_pgdir[PDX(VPT)]   = e->env_cr3 ;
       e->env_pgdir[PDX(UVPT)]  = e->env_cr3 ;
       
       (Hint2:为什么这里要让pgdir[PDX(UVPT)]=e->env_cr3?还记得windows的自映射方案吗?)
       
       return 0;
}
```
    
    其实这个函数并不需要我们实现，但是我还是想讲一讲这个函数的一些有意思的地方。

　　我们知道，每一个进程都有4G的逻辑地址可以访问，我们所熟知的系统不管是Linux还是Windows系统，都可以支持3G/1G模式或者2G/2G模式。3G/1G模式即满32位的进程地址空间中，用户态占3G，内核态占1G。这些情况在进入内核态的时候叫做陷入内核，因为即使进入了内核态，还处在同一个地址空间中，并不切换CR3寄存器。但是！还有一种模式是4G/4G模式，内核单独占有一个4G的地址空间，所有的用户进程独享自己的4G地址空间，这种模式下，在进入内核态的时候，叫做切换到内核，因为需要切换CR3寄存器，所以进入了不同的地址空间！

　　而我们这次实验，根据./include/mmu.h里面的布局来说，我们其实就是2G/2G模式，用户态占用2G，内核态占用2G。所以记住，我们在用户进程开启后，访问内核地址不需要切换`cr3`寄存器！其实这个布局模式也很好地解释了为什么我们需要把boot_pgdir里的内容拷到我们的e->env_pgdir中，在我们的实验中，对于不同的进程而言，其虚拟地址ULIM以上的地方，映射关系都是一样的。这是因为这2G虚拟地址与物理地址的对应，不是由进程管理的，是由内核管理的。

　　另外一点有意思的地方(Hint1)不知大家注意到没有，UTOP~ULIM明明是属于User的区域，却还是把内核这部分映射到了User区，我们看看mmu.h里面的图，会觉得很有意思：
```C
o            ULIM     -----> +----------------------------+-------------0x80000000    
o                       　　      |         User VPT           　　  |     PDMAP                　　   
o            UVPT     -----> +----------------------------+-------------0x7fc00000   
o                      　　       |         PAGES              　　  |     PDMAP                 　　  
o            UPAGES   -----> +----------------------------+-------------0x7f800000  
o                      　　       |         ENVS               　　  |     PDMAP                 　　  
o        UTOP,UENVS   -----> +----------------------------+-------------0x7f400000    
o          UXSTACKTOP -/          |     user exception stack         |     BY2PG                         
o                      　　　 +----------------------------+------------0x7f3ff000      
o                      　　　　   |       Invalid memory             |     BY2PG                 　　  
o            USTACKTOP ----> +----------------------------+------------ 0x7f3fe000     
o                         　　　　|     normal user stack            |     BY2PG                 　　 
o                     　　　  +----------------------------+------------0x7f3fd000    
a                      　　　　   |                            　　  |
```  
`UVPT`即`User Virtual Page Table`,根据字面意思理解,它是用户页表域的那4M虚拟空间。
其余内容稍候补充...

##env_alloc##
```C
int env_alloc(struct Env **new, u_int parent_id)
{
        int r;
        /*precondtion: env_init has been called before this function*/
        /*1. get a new Env from env_free_list*/
        struct Env *currentE;
        currentE = LIST_FIRST(&env_free_list);
        /*2. call some function(has been implemented) to intial kernel memory layout for this new Env.
         *hint:please read this c file carefully, this function mainly map the kernel address to this new Env address*/
        if((r=env_setup_vm(currentE))<0)
                return r;
        /*3. initial every field of new Env to appropriate value*/
        currentE->env_id = mkenvid(currentE);
        currentE->env_parent_id = parent_id;
        currentE->env_status = ENV_NOT_RUNNABLE;
        /*4. focus on initializing env_tf structure, located at this new Env. especially the sp register,
         * CPU status and PC register(the value of PC can refer the comment of load_icode function)*/
       currentE->env_tf.regs[29] = USTACKTOP;
       
        (Hint1:为什么pc是UTEXT+0xb0?改成其他可行吗?)
        
        currentE->env_tf.pc = UTEXT + 0xb0;
        
        (Hint2:R3000的SR寄存器与MIPSR4000及以后的版本略有不同,为什么这里是0x10001004?试着改成0x10001005,仔细观察后6位)
        
        currentE->env_tf.cp0_status = 0x10001004;
        /*5. remove the new Env from Env free list*/
        LIST_REMOVE(currentE,env_link);
        *new = currentE;
        return 0;
}
```

```C
static void
load_icode(struct Env *e, u_char *binary, u_int size)
{
        int r;
        u_long currentpg,endpg;

        currentpg = UTEXT;
        endpg = currentpg + ROUND(size,BY2PG);
        /*precondition: we have a valid Env object pointer e, a valid binary pointer pointing to some valid
        machine code(you can find them at $WORKSPACE/init/ directory, such as code_a_c, code_b_c,etc), which can
         *be executed at MIPS, and its valid size */
        while(currentpg < endpg){
             struct Page *page;
             if((r= page_alloc(&page))<0)
                return;
             if((r= page_insert(e->env_pgdir,page,currentpg,PTE_V))<0)
                return;
             bzero(page2kva(page),BY2PG);
             
            (Hint:这里不按BY2PG为单位分配可以不可以,什么情况下可以,什么情况下不可以?会有什么样的问题出现?)
            
             (Hint:这里为什么是用page2kva(page),而不是用UTEXT段copy?)
             bcopy((void *)binary,page2kva(page),BY2PG);
             binary += BY2PG;
             currentpg +=BY2PG;
        }
        /*1. copy the binary code(machine code) to Env address space(start from UTEXT to high address), it may call some auxiliare function
        (eg,page_insert or bcopy.etc)*/
        struct Page *stack;
        page_alloc(&stack);
        
        (Hint:PTE_R代表什么意义?这里不用PTE_R设权限会在什么时候出现问题?)
        
        page_insert(e->env_pgdir,stack,(USTACKTOP-BY2PG),PTE_V|PTE_R);
        /*2. make sure PC(env_tf.pc) point to UTEXT + 0xb0, this is very import, or your code is not executed correctly when your process(namely Env) is dispatched by CPU*/
        assert(e->env_tf.pc == UTEXT+0xb0);
        e->env_status = ENV_RUNNABLE;
}

```
```C
int
page_insert(Pde *pgdir, struct Page *pp, u_long va, u_int perm)
{       // Fill this function in
        u_int PERM;
        Pte *pgtable_entry;
        PERM = perm | PTE_V;
        pgdir_walk(pgdir, va, 0, &pgtable_entry);
        
        if(pgtable_entry!=0 &&(*pgtable_entry & PTE_V)!=0)
                if(pa2page(*pgtable_entry)!=pp) page_remove(pgdir,va);
                else
                {
                
                        (Hint:为什么多次出现了tlb_invalidate函数?这个函数作用是什么?)
                        
                        tlb_invalidate(pgdir, va);
                        
                        (Hint:这里为什么要强调权限位的重新设置?)
                        
                        *pgtable_entry = (page2pa(pp)|PERM);
                        return 0;
                }
        tlb_invalidate(pgdir, va);
        
        if(pgdir_walk(pgdir, va, 1, &pgtable_entry)!=0){
                return -E_NO_MEM;
        }
        *pgtable_entry = (page2pa(pp)|PERM);
        pp->pp_ref++;
        return 0;
}
```
