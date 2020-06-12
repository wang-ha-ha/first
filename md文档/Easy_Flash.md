##### Easy_Flash

[EasyFlash](https://github.com/armink/EasyFlash)是一款开源的轻量级嵌入式Flash存储器库，方便开发者更加轻松的实现基于Flash存储器的常见应用开发。非常适合智能家居、可穿戴、工控、医疗、物联网等需要断电存储功能的产品，资源占用极低，支持各种 MCU 片上存储器。该库主要包括 **三大实用功能** ：

- **ENV** 快速保存产品参数，支持 **写平衡（磨损平衡）** 及 **掉电保护** 功能

EasyFlash不仅能够实现对产品的 **设定参数** 或 **运行日志** 等信息的掉电保存功能，还封装了简洁的 **增加、删除、修改及查询** 方法， 降低了开发者对产品参数的处理难度，也保证了产品在后期升级时拥有更好的扩展性。让Flash变为NoSQL（非关系型数据库）模型的小型键值（Key-Value）存储数据库。

- **IAP** 在线升级再也不是难事儿

该库封装了IAP(In-Application Programming)功能常用的接口，支持CRC32校验，同时支持Bootloader及Application的升级。

- **Log** 无需文件系统，日志可直接存储在Flash上

非常适合应用在小型的不带文件系统的产品中，方便开发人员快速定位、查找系统发生崩溃或死机的原因。同时配合[EasyLogger](https://github.com/armink/EasyLogger)(开源的超轻量级、高性能C日志库，它提供与EasyFlash的无缝接口)一起使用，轻松实现C日志的Flash存储功能。

##### 带着问题去学习

- 上电初始化从flash读到内存的过程
- 怎么读写变量
- 在内存中的缓存是什么数据结构，做用是什么
- 分扇区添加头部的作用
- 重写接口函数时，擦和写函数需要注意页吗
- 擦写掉电如何恢复现场（掉电保护）

##### 初始化过程

###### easyflash_init

```c
EfErrCode easyflash_init(void) {    
    extern EfErrCode ef_port_init(ef_env const **default_env, size_t *default_env_size);    
    extern EfErrCode ef_env_init(ef_env const *default_env, size_t default_env_size);
    extern EfErrCode ef_iap_init(void);
    extern EfErrCode ef_log_init(void);

    size_t default_env_set_size = 0;
    const ef_env *default_env_set;
    EfErrCode result = EF_NO_ERR;
	//使用者写的初始函数，主要用于初始化相关硬件资源和初始环境变量
    result = ef_port_init(&default_env_set, &default_env_set_size);
#ifdef EF_USING_ENV
    //EasyFlash的初始化函数
    if (result == EF_NO_ERR) {
        result = ef_env_init(default_env_set, default_env_set_size);
    }
#endif
#ifdef EF_USING_IAP
    if (result == EF_NO_ERR) {
        result = ef_iap_init();
    }
#endif
#ifdef EF_USING_LOG
    if (result == EF_NO_ERR) {
        result = ef_log_init();
    }
#endif
    return result;
}
```

###### ef_env_init

```c
EfErrCode ef_env_init(ef_env const *default_env, size_t default_env_size) {
    EfErrCode result = EF_NO_ERR;

#ifdef EF_ENV_USING_CACHE
    size_t i;
#endif

    EF_ASSERT(default_env);
    EF_ASSERT(ENV_AREA_SIZE);
    /* must be aligned with erase_min_size */
    EF_ASSERT(ENV_AREA_SIZE % EF_ERASE_MIN_SIZE == 0);
    /* sector number must be greater than or equal to 2 */
    EF_ASSERT(SECTOR_NUM >= 2);
    /* must be aligned with write granularity */
    EF_ASSERT((EF_STR_ENV_VALUE_MAX_SIZE * 8) % EF_WRITE_GRAN == 0);

    if (init_ok) {
        return EF_NO_ERR;
    }
//初始化内存缓存
#ifdef EF_ENV_USING_CACHE
    for (i = 0; i < EF_SECTOR_CACHE_TABLE_SIZE; i++) {
        sector_cache_table[i].addr = FAILED_ADDR;
    }
    for (i = 0; i < EF_ENV_CACHE_TABLE_SIZE; i++) {
        env_cache_table[i].addr = FAILED_ADDR;
    }
#endif /* EF_ENV_USING_CACHE */
	
    //给环境变量赋开始值
    env_start_addr = EF_START_ADDR;
    default_env_set = default_env;
    default_env_set_size = default_env_size;

    EF_DEBUG("ENV start address is 0x%08X, size is %d bytes.\n", EF_START_ADDR, ENV_AREA_SIZE);
	
    //检测flash中扇区头部的有效性，如果全部无效，重新导入环境变量
    result = ef_load_env();

#ifdef EF_ENV_AUTO_UPDATE
    if (result == EF_NO_ERR) {
        env_auto_update();
    }
#endif

    if (result == EF_NO_ERR) {
        init_ok = true;
    }

    return result;
}
```

###### ef_load_env

```c
EfErrCode ef_load_env(void)
{
    EfErrCode result = EF_NO_ERR;
    struct env_meta_data env;
    struct sector_meta_data sector;
    size_t check_failed_count = 0;

    in_recovery_check = true;
    /* check all sector header */
    sector_iterator(&sector, SECTOR_STORE_UNUSED, &check_failed_count, NULL, check_sec_hdr_cb, false);
    /* all sector header check failed */
    if (check_failed_count == SECTOR_NUM) {
        EF_INFO("Warning: All sector header check failed. Set it to default.\n");
        ef_env_set_default();
    }
    
    /* lock the ENV cache */
    ef_port_env_lock();
    /* check all sector header for recovery GC */
    sector_iterator(&sector, SECTOR_STORE_UNUSED, NULL, NULL, check_and_recovery_gc_cb, false);

__retry:
    /* check all ENV for recovery */
    env_iterator(&env, NULL, NULL, check_and_recovery_env_cb);
    if (gc_request) {
        gc_collect();
        goto __retry;
    }

    in_recovery_check = false;

    /* unlock the ENV cache */
    ef_port_env_unlock();

    return result;
}
```

##### 缓存的数据结构

​		设计的不复杂，直接使用的数组。

```c
struct env_cache_node {
    uint16_t name_crc;          /**< ENV name's CRC32 low 16bit value */
    uint16_t active;            /**< ENV node access active degree */
    uint32_t addr;              /**< ENV node address */
};

struct sector_cache_node {
    uint32_t addr;              /**< sector start address */
    uint32_t empty_addr;        /**< sector empty address */
};
#ifdef EF_ENV_USING_CACHE
/* ENV cache table */
struct env_cache_node env_cache_table[EF_ENV_CACHE_TABLE_SIZE] = { 0 };
/* sector cache table, it caching the sector info which status is current using */
struct sector_cache_node sector_cache_table[EF_SECTOR_CACHE_TABLE_SIZE] = { 0 };
#endif /* EF_ENV_USING_CACHE */
```

##### 扇区的遍历

###### 		sector_iterator

​		主要是 sector_iterator 这个函数去遍历，通过传入不同的回调函数做不同的处理

```c
/*
*     sector 传入扇区操作结构体，需要提前申请好内存
*	  status 对应需要执行操作的状态
*	  arg1、arg2 给回调函数的参数
*	  callback 回调函数
*     traversal 是否需要全部遍历
*/

static void sector_iterator(sector_meta_data_t sector, sector_store_status_t status, void *arg1, void *arg2,
        bool (*callback)(sector_meta_data_t sector, void *arg1, void *arg2), bool traversal_env) {
    uint32_t sec_addr;

    /* search all sectors */
    sector->addr = FAILED_ADDR;
    while ((sec_addr = get_next_sector_addr(sector)) != FAILED_ADDR) {
        //读扇区头部信息
        read_sector_meta_data(sec_addr, sector, false);
        if (status == SECTOR_STORE_UNUSED || status == sector->status.store) {
            if (traversal_env) {
                read_sector_meta_data(sec_addr, sector, traversal_env);
            }
            /* iterator is interrupted when callback return true */
            if (callback && callback(sector, arg1, arg2)) {
                return;
            }
        }
    }
}
```

​		sector_iterator 回调函数

```c
/*
*   sector_iterator 检测扇区头部的回调函数
*/
static bool check_sec_hdr_cb(sector_meta_data_t sector, void *arg1, void *arg2)
{
    if (!sector->check_ok) {
        size_t *failed_count = arg1;

        EF_INFO("Warning: Sector header check failed. Format this sector (0x%08x).\n", sector->addr);
        (*failed_count) ++;
        format_sector(sector->addr, SECTOR_NOT_COMBINED);
    }

    return false;
}
```

```c
/*
*   sector_iterator 统计没有使用的扇区数量
*/
static bool sector_statistics_cb(sector_meta_data_t sector, void *arg1, void *arg2)
{
    size_t *empty_sector = arg1, *using_sector = arg2;

    if (sector->check_ok && sector->status.store == SECTOR_STORE_EMPTY) {
        (*empty_sector)++;
    } else if (sector->check_ok && sector->status.store == SECTOR_STORE_USING) {
        (*using_sector)++;
    }

    return false;
}
```

```c
/*
*   sector_iterator 检测到有剩余空间的扇区
*/
static bool alloc_env_cb(sector_meta_data_t sector, void *arg1, void *arg2)
{
    size_t *env_size = arg1;
    uint32_t *empty_env = arg2;

    /* 1. sector has space
     * 2. the NO dirty sector
     * 3. the dirty sector only when the gc_request is false */
    if (sector->check_ok && sector->remain > *env_size
            && ((sector->status.dirty == SECTOR_DIRTY_FALSE)
                    || (sector->status.dirty == SECTOR_DIRTY_TRUE && !gc_request))) {
        *empty_env = sector->empty_env;
        return true;
    }

    return false;
}
```

```c
static bool gc_check_cb(sector_meta_data_t sector, void *arg1, void *arg2)
{
    size_t *empty_sec = arg1;

    if (sector->check_ok) {
        *empty_sec = *empty_sec + 1;
    }

    return false;

}

static bool do_gc(sector_meta_data_t sector, void *arg1, void *arg2)
{
    struct env_meta_data env;

    if (sector->check_ok && (sector->status.dirty == SECTOR_DIRTY_TRUE || sector->status.dirty == SECTOR_DIRTY_GC)) {
        uint8_t status_table[DIRTY_STATUS_TABLE_SIZE];
        /* change the sector status to GC */
        write_status(sector->addr + SECTOR_DIRTY_OFFSET, status_table, SECTOR_DIRTY_STATUS_NUM, SECTOR_DIRTY_GC);
        /* search all ENV */
        env.addr.start = FAILED_ADDR;
        while ((env.addr.start = get_next_env_addr(sector, &env)) != FAILED_ADDR) {
            read_env(&env);
            if (env.crc_is_ok && (env.status == ENV_WRITE || env.status == ENV_PRE_DELETE)) {
                /* move the ENV to new space */
                if (move_env(&env) != EF_NO_ERR) {
                    EF_DEBUG("Error: Moved the ENV (%.*s) for GC failed.\n", env.name_len, env.name);
                }
            }
        }
        format_sector(sector->addr, SECTOR_NOT_COMBINED);
        EF_DEBUG("Collect a sector @0x%08X\n", sector->addr);
    }

    return false;
}
```



###### 		get_next_sector_addr

​		拿到下一个扇区的地址

```c
static uint32_t get_next_sector_addr(sector_meta_data_t pre_sec)
{
    uint32_t next_addr;

    if (pre_sec->addr == FAILED_ADDR) {
        return env_start_addr;
    } else {
        /* check ENV sector combined */
        if (pre_sec->combined == SECTOR_NOT_COMBINED) {
            next_addr = pre_sec->addr + SECTOR_SIZE;
        } else {
            next_addr = pre_sec->addr + pre_sec->combined * SECTOR_SIZE;
        }
        /* check range */
        if (next_addr < env_start_addr + ENV_AREA_SIZE) {
            return next_addr;
        } else {
            /* no sector */
            return FAILED_ADDR;
        }
    }
}
```

###### 	format_sector

​		格式化扇区

```c
static EfErrCode format_sector(uint32_t addr, uint32_t combined_value)
{
    EfErrCode result = EF_NO_ERR;
    struct sector_hdr_data sec_hdr;

    EF_ASSERT(addr % SECTOR_SIZE == 0);

    result = ef_port_erase(addr, SECTOR_SIZE);
    if (result == EF_NO_ERR) {
        /* initialize the header data */
        memset(&sec_hdr, 0xFF, sizeof(struct sector_hdr_data));
        set_status(sec_hdr.status_table.store, SECTOR_STORE_STATUS_NUM, SECTOR_STORE_EMPTY);
        set_status(sec_hdr.status_table.dirty, SECTOR_DIRTY_STATUS_NUM, SECTOR_DIRTY_FALSE);
        sec_hdr.magic = SECTOR_MAGIC_WORD;
        sec_hdr.combined = combined_value;
        sec_hdr.reserved = 0xFFFFFFFF;
        /* save the header */
        result = ef_port_write(addr, (uint32_t *)&sec_hdr, sizeof(struct sector_hdr_data));

#ifdef EF_ENV_USING_CACHE
        /* delete the sector cache */
        update_sector_cache(addr, addr + SECTOR_SIZE);
#endif /* EF_ENV_USING_CACHE */
    }

    return result;
}
```



##### 垃圾回收函数

```c
static void gc_collect(void)
{
    struct sector_meta_data sector;
    size_t empty_sec = 0;

    /* GC check the empty sector number */
    sector_iterator(&sector, SECTOR_STORE_EMPTY, &empty_sec, NULL, gc_check_cb, false);

    /* do GC collect */
    EF_DEBUG("The remain empty sector is %d, GC threshold is %d.\n", empty_sec, EF_GC_EMPTY_SEC_THRESHOLD);
    if (empty_sec <= EF_GC_EMPTY_SEC_THRESHOLD) {
        sector_iterator(&sector, SECTOR_STORE_UNUSED, NULL, NULL, do_gc, false);
    }

    gc_request = false;
}
```



##### 创建、读取、修改环境变量