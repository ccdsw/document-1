# CFQ代码逻辑梳理

## 1. 结构体介绍
### 1.1 iosched_cfq
定义一个电梯调度器--cfq, 实例化一个CFQ调度器.
```
static struct elevator_type iosched_cfq = {                                                                                                                              
        .ops = {                                                                                                                                                         
                .elevator_merge_fn =            cfq_merge,   //request和bio合并的时候调用 
                .elevator_merged_fn =           cfq_merged_request, //看到这个成员的ed没？这个merged是指已经合并了的request
                .elevator_merge_req_fn =        cfq_merged_requests, //这个是两个request合并的时候调用,  调整在RB树种的位置？                            
                .elevator_allow_bio_merge_fn =  cfq_allow_bio_merge,    //request和bio能否合并的时候调用                                   
                .elevator_allow_rq_merge_fn =   cfq_allow_rq_merge,              
                .elevator_bio_merged_fn =       cfq_bio_merged,                                                                                                          
                .elevator_dispatch_fn =         cfq_dispatch_requests,  //用当前的request来填充dispatch queue，一旦dispatch，调度器就不能控制这些io请求了，
                .elevator_add_req_fn =          cfq_insert_request,   //增加一个新请求到调度器
                .elevator_activate_req_fn =     cfq_activate_request,  //这个相当于让block层看到这个request
                .elevator_deactivate_req_fn =   cfq_deactivate_request, //这个就是block驱动再推迟这个请求
                .elevator_completed_req_fn =    cfq_completed_request, // req结束之后调用
                .elevator_former_req_fn =       elv_rb_former_request,  //查找给定请求的前一个请求
                .elevator_latter_req_fn =       elv_rb_latter_request,  //查找给定请求的后一个请求，在进行request合并时很有用
                .elevator_init_icq_fn =         cfq_init_icq,  //icq的构造函数
                .elevator_exit_icq_fn =         cfq_exit_icq,  //icq的析构函数
                .elevator_set_req_fn =          cfq_set_request, //创建请求时填充req的一些必要字段
                .elevator_put_req_fn =          cfq_put_request, //释放请求时填充req的一些必要字段和处理
                .elevator_may_queue_fn =        cfq_may_queue,  // 如果调度器允许在超过queue的limit情况下依然插入请求，return true
                .elevator_init_fn =             cfq_init_queue,   //队列释放时调用，相当于析构函数                         
                .elevator_exit_fn =             cfq_exit_queue, //队列释放时调用，相当于析构函数
                .elevator_registered_fn =       cfq_registered_queue,                                                                                                    
        },                                                                                                                                                               
        .icq_size       =       sizeof(struct cfq_io_cq),                                                                                                                
        .icq_align      =       __alignof__(struct cfq_io_cq),                                                                                                           
        .elevator_attrs =       cfq_attrs,                                                                                                                               
        .elevator_name  =       "cfq",                                                                                                                                   
        .elevator_owner =       THIS_MODULE,                                                                                                                             
};
```

### 1.2 cfq_data
`cfq_data`对应一个块设备
```
/*
 * Per block device queue structure
 */
struct cfq_data {
        struct request_queue *queue;    //指向块设备对应的request_queue                        
        /* Root service tree for cfq_groups */                                                                                                                           
        struct cfq_rb_root grp_service_tree; //所有待调度的cfq_group都被加入到该红黑树，管理cfq_group的rb_node成员。
        struct cfq_group *root_group; //所有cggroup的根cgroup
                                                                                                                                                                         
        /*                                                                                                                                                               
         * The priority currently being served                                                                                                                           
         */                                                                                                                                                              
        enum wl_class_t serving_wl_class;  //当前服务的BE_WORKLOAD = 0,RT_WORKLOAD = 1,IDLE_WORKLOAD = 2,CFQ_PRIO_NR=3,
        enum wl_type_t serving_wl_type;  //当前服务的cfq_group
        u64 workload_expires;                                                                                                                                            
        struct cfq_group *serving_group;  //当前服务的cfq_group
                                                                                                                                                                         
        /*                                                                                                                                                               
         * Each priority tree is sorted by next_request position.  These                                                                                                 
         * trees are used when determining if two or more queues are                                                                                                     
         * interleaving requests (see cfq_close_cooperator).                                                                                                             
         */                                                                                                                                                              
        struct rb_root prio_trees[CFQ_PRIO_LISTS]; // CFQ_PRIO_LISTS为8，8个优先级的红黑树，所有优先级为rt或be的进程的同步请求队列，都会根据优先级添加到对应的红黑树
                                                                                                                                                                         
        unsigned int busy_queues; // 该cfqd中有多少个队列在等待调度
        unsigned int busy_sync_queues; // 该cfqd中有多少个同步队列在等待调度
                                                                                                                                                                         
        int rq_in_driver;  // 在驱动中，也就是调度层已经下发，但还没有收到结果的request数
        int rq_in_flight[2]; // 已经下发还未收到结果的同步和异步的io个数
        /*                                                                                                                                                               
         * queue-depth detection  
         */                                                                                                                                                              
        int rq_queued;  // queue深度的一些成员，这个是积压的request的个数
        int hw_tag;                                                                                                                                                      
        /*                                                                                                                                                               
         * hw_tag can be                                                                                                                                                 
         * -1 => indeterminate, (cfq will behave as if NCQ is present, to allow better detection)                                                                        
         *  1 => NCQ is present (hw_tag_est_depth is the estimated max depth)                                                                                            
         *  0 => no NCQ                                                                                                                                                  
         */                                                                                                                                                              
        int hw_tag_est_depth;                                                                                                                                            
        unsigned int hw_tag_samples;                                                                                                                                     
                                                                                                                                                                         
        /*                                                                                                                                                               
         * idle window management                                                                                                                                        
         */                                                                                                                                                              
        struct hrtimer idle_slice_timer;                                                                                                                                 
        struct work_struct unplug_work;                                                                                                                                  
                                                                                                                                                                         
        struct cfq_queue *active_queue; // 指向当前服务的队列，因为cfq_data是每个使用cfq调度算法的设备的数据结构，对每个时刻而言，每个cfq_data只服务一个cfq_queue
        struct cfq_io_cq *active_cic; // 当前活动的io上下文，和active_queue一样，都算是缓存    cic是 cfq_io_context 的简写
                                                                                                                                                                         
        sector_t last_position;                                                                                                                                          
                                                                                                                                                                         
        /*                                                                                                                                                               
         * tunables, see top of file                                                                                                                                     
         */                                                                                                                                                              
        unsigned int cfq_quantum;  //用于计算在一个队列的时间片内，最多发放多少个请求到底层的块设备, cfq_quantum = 8;
        unsigned int cfq_back_penalty; // cfq_back_penalty = 2;
        unsigned int cfq_back_max; // cfq_back_max = 16 * 1024;
        unsigned int cfq_slice_async_rq;  // cfq_slice_async_rq = 2;
        unsigned int cfq_latency; // cfqd->cfq_latency = 1,可以设置为0
        u64 cfq_fifo_expire[2]; //同步、异步请求的响应期限时间,cfq_fifo_expire[2] = { HZ / 4, HZ / 8 };                                                                  
        u64 cfq_slice[2]; // 同步、异步请求队列的时间片长度,cfq_slice_async = HZ / 25; cfq_slice_sync = HZ / 10;
        u64 cfq_slice_idle; //idle的时间片,cfq_slice_idle = HZ / 125;
        u64 cfq_group_idle; // cfq_group_idle = HZ / 125;
        u64 cfq_target_latency; // cfq_target_latency = HZ * 3/10; /* 300 ms */
                                                                                                                                                                         
        /*                                                                                                                                                               
         * Fallback dummy cfqq for extreme OOM conditions                                                                                                                
         */                                                                                                                                                              
        struct cfq_queue oom_cfqq; // 在极端的oom的情况下，因为alloc cfq_queue失败，则使用这个queue，相当于备用的
                                                                                                                                                                         
        u64 last_delayed_sync;                                                                                                                                           
};                                                                                                                                                                       

```
### 1.3 cfq_queue
```
/*                                                                                                                                                                       
 * Per process-grouping structure                                                                                                                                        
 */                                                                                                                                                                      
struct cfq_queue {                                                                                                                                                       
        /* reference count */                                                                                                                                            
        int ref;                                                                                                                                                         
        /* various state flags, see below */                                                                                                                             
        unsigned int flags;                                                                                                                                              
        /* parent cfq_data */                                                                                                                                            
        struct cfq_data *cfqd;    // 指向队列所属的cfq_data                  
        /* service_tree member */                                                                                                                                        
        struct rb_node rb_node; // 用于将该队列插入service_tree,就是红黑树的node，注意和sort_list相区别，sort_list是管理的对象是request
        /* service_tree key */                                                                                                                                           
        u64 rb_key;// 红黑树节点关键值，用于确定队列在service_tree中的位置，该值要综合jiffies，进程的io优先级等考虑
        /* prio tree member */                                                                                                                                           
        struct rb_node p_node; // 用于将队列插入对应优先级的prio_tree
        /* prio tree root we belong to, if any */                                                                                                                        
        struct rb_root *p_root; // 对应的prio_tree树根，就是当前cfq_q归属于cfqd中的prio_trees的节点的地址
        /* sorted list of pending requests */                                                                                                                            
        struct rb_root sort_list;// 组织队列内的请求用的红黑树，按请求的起始扇区进行排序，便于查找某个扇区以及合并请求
        /* if fifo isn't expired, next request to serve */                                                                                                               
        struct request *next_rq; //红黑树中下一个要处理的请求
        /* requests queued in sort_list */                                                                                                                               
        int queued[2]; //目前管理的同步和异步的request的数量，数组1是同步io请求的数量
        /* currently allocated requests */                                                                                                                               
        int allocated[2]; // 读写两个方向的分配的request的数量，allocated包含dispatch+queued
        /* fifo list of requests in sort_list */                                                                                                                         
        struct list_head fifo; // fifo的list，pending requests形成的fifo链表，fifo主要按时间来管理，主要防止队列内某些request饿死。
        //为什么既用fifo来管理request，又用rb_tree来管理呢？其实内核中很多这样的管理方式，比如说我们的线性区也有多个管理结构，
        //deadline io scheduler也有类似的，主要是不同维度来管理

        /* time when queue got scheduled in to dispatch first request. */                                                                                                
        u64 dispatch_start;   // 当被选中来调度时的时间
        u64 allocated_slice; // 被选中时分配的时间片
        u64 slice_dispatch; // 时间片内给对应的request queue下发的请求数，这个在一个调度周期内有效，下次选择到这个cfq_q会清零           
        /* time when first request from queue completed and slice started. */                                                                                            
        u64 slice_start;  // 指定时间片什么时候开始                            
        u64 slice_end;   // 指明时间片何时消耗完
        s64 slice_resid;                                                                                                                                                 
                                                                                                                                                                         
        /* pending priority requests */                                                                                                                                  
        int prio_pending;                                                                                                                                                
        /* number of requests that are on the dispatch list or inside driver */                                                                                          
        int dispatched;  // 该cfq_q总共下发的request总数，这些是驱动待处理的请求
                                                                                                                                                                         
        /* io prio of this group */                                                                                                                                      
        unsigned short ioprio, org_ioprio;   
        unsigned short ioprio_class, org_ioprio_class;                                                                                                                   
                                                                                                                                                                         
        pid_t pid; // 该cfq_queue归属的进程的pid
                                                                                                                                                                         
        u32 seek_history;                                                                                                                                                
        sector_t last_request_pos;                                                                                                                                       
                                                                                                                                                                         
        struct cfq_rb_root *service_tree;   // 指向cfq_data对应的cfq_rb_tree
        struct cfq_queue *new_cfqq;                                                                                                                                      
        struct cfq_group *cfqg; // cfq_queue对应的cgroup,主要是进程信息，看组内的大家怎么分时间片，异步队列的话，这个就指向cfq_data->root_group
        /* Number of sectors dispatched from queue in single dispatch round */                                                                                           
        unsigned long nr_sectors; // 调度周期内下发的扇区总数
};
```

### 1.4 cfq_group

```
/* This is per cgroup per device grouping structure */
struct cfq_group {
        /* must be the first member */                                                                                                                                   
        struct blkg_policy_data pd;                                                                                                                                      
                                                                                                                                                                         
        /* group service_tree member */                                                                                                                                  
        struct rb_node rb_node;                                                                                                                                          
                                                                                                                                                                         
        /* group service_tree key */                                                                                                                                     
        u64 vdisktime;                                                                                                                                                   
                                                                                                                                                                         
        /*                                                                                                                                                               
         * The number of active cfqgs and sum of their weights under this                                                                                                
         * cfqg.  This covers this cfqg's leaf_weight and all children's                                                                                                 
         * weights, but does not cover weights of further descendants.                                                                                                   
         *                                                                                                                                                               
         * If a cfqg is on the service tree, it's active.  An active cfqg                                                                                                
         * also activates its parent and contributes to the children_weight                                                                                              
         * of the parent.                                                                                                                                                
         */                                                                                                                                                              
        int nr_active;                                                                                                                                                   
        unsigned int children_weight;                                                                                                                                    
                                                                                                                                                                         
        /*                                                                                                                                                               
         * vfraction is the fraction of vdisktime that the tasks in this                                                                                                 
         * cfqg are entitled to.  This is determined by compounding the                                                                                                  
         * ratios walking up from this cfqg to the root.                                                                                                                 
         *                                                                                                                                                               
         * It is in fixed point w/ CFQ_SERVICE_SHIFT and the sum of all                                                                                                  
         * vfractions on a service tree is approximately 1.  The sum may                                                                                                 
         * deviate a bit due to rounding errors and fluctuations caused by                                                                                               
         * cfqgs entering and leaving the service tree.                                                                                                                  
         */                                                                                                                                                              
        unsigned int vfraction;
                                                                                                                                          
        /*                                                                                                                                                               
         * There are two weights - (internal) weight is the weight of this                                                                                               
         * cfqg against the sibling cfqgs.  leaf_weight is the wight of                                                                                                  
         * this cfqg against the child cfqgs.  For the root cfqg, both                                                                                                   
         * weights are kept in sync for backward compatibility.                                                                                                          
         */                                                                                                                                                              
        unsigned int weight;                                                                                                                                             
        unsigned int new_weight;                                                                                                                                         
        unsigned int dev_weight;                                                                                                                                         
                                                                                                                                                                         
        unsigned int leaf_weight;                                                                                                                                        
        unsigned int new_leaf_weight;                                                                                                                                    
        unsigned int dev_leaf_weight;                                                                                                                                    
                                                                                                                                                                         
        /* number of cfqq currently on this group */                                                                                                                     
        int nr_cfqq;                                                                                                                                                     
                                                                                                                                                                         
        /*                                                                                                                                                               
         * Per group busy queues average. Useful for workload slice calc. We                                                                                             
         * create the array for each prio class but at run time it is used                                                                                               
         * only for RT and BE class and slot for IDLE class remains unused.                                                                                              
         * This is primarily done to avoid confusion and a gcc warning.                                                                                                  
         */                                                                                                                                                              
        unsigned int busy_queues_avg[CFQ_PRIO_NR];                                                                                                                       
        /*                                                                                                                                                               
         * rr lists of queues with requests. We maintain service trees for                                                                                               
         * RT and BE classes. These trees are subdivided in subclasses                                                                                                   
         * of SYNC, SYNC_NOIDLE and ASYNC based on workload type. For IDLE                                                                                               
         * class there is no subclassification and all the cfq queues go on                                                                                              
         * a single tree service_tree_idle.                                                                                                                              
         * Counts are embedded in the cfq_rb_root                                                                                                                        
         */                                                                                                                                                              
        struct cfq_rb_root service_trees[2][3];                                                                                                                          
        struct cfq_rb_root service_tree_idle;  

        u64 saved_wl_slice;                                                                                                                                              
        enum wl_type_t saved_wl_type;                                                                                                                                    
        enum wl_class_t saved_wl_class;                                                                                                                                  
                                                                                                                                                                         
        /* number of requests that are on the dispatch list or inside driver */                                                                                          
        int dispatched;                                                                                                                                                  
        struct cfq_ttime ttime;                                                                                                                                          
        struct cfqg_stats stats;        /* stats for this cfqg */                                                                                                        
                                                                                                                                                                         
        /* async queue for each priority case */                                                                                                                         
        struct cfq_queue *async_cfqq[2][IOPRIO_BE_NR];                                                                                                                   
        struct cfq_queue *async_idle_cfqq;                                                                                                                               
                                                                                                                                                                         
};
```

**cfq代码中常见的简写：
cic是 cfq_io_context 的简写
cfqq 是cfq_queue  的简写
cfqd是cfq_data 的简写
be是  best effort 的简写
rt是 real time 的简写
其中有三种class: idle(3), best effort(2), real time(1)，具体用法man ionice
real time 和 best effort 内部都有 0-7 一共8个优先级，对于real time而言，由于优先级高，有可能会饿死其他进程，对于 best effort 而言，2.6.26之后的内核如不指定io priority，那就有io priority = cpu nice**

![image](http://note.youdao.com/noteshare?id=5071ac19479d0e1f10e99af34204f84d&sub=wcp15841049656104)
