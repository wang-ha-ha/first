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

