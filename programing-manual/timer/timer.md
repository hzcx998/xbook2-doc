# 时钟管理

在操作系统中，时间的最小单位是时钟节拍（clock ticks），基于这个，我们可以使用定时器来做一些想做的事情，除此之外，也可以用来计算墙上时间，也就是计算现在的时间，几时几分几秒。读完本章，我们将了解时钟节拍如何产生，并学会使用xbook2的定时器。

## 时钟节拍

时钟节拍就是，可以对某个时钟硬件进行编程，控制它1s产生多少次时钟中断，我们可以用一个变量`sys_ticks`来记录这个次数，可以是10次，也可以是100次，甚至是1000次。那么这个就是时钟节拍。xbook2中通过`HZ`这个变量来表明1s可以产生的时钟节拍数量。

## 时钟节拍的实现方式

当时钟中断产生时，我们将`sys_ticks+=1`，我们的定时器和墙上时间就是基于sys_ticks来的。

```c
static int clock_handler(irqno_t irq, void *data)
{
	systicks++;
    timer_ticks++;
	softirq_active(TIMER_SOFTIRQ);
	softirq_active(SCHED_SOFTIRQ);
    return 0;
}
```

## 获取时钟节拍

可以通过`sys_get_ticks`来获取当前的ticks值。

```c
clock_t sys_get_ticks()
{
    return systicks;
}
```

## 定时器管理

定时器就是你可以设置一个时间点，当到达这个时间点时，将会产生超时，你可以在超时点做一些你想做的事情。

```c
struct timer_struct;
// callback(timer, arg)
typedef void (*timer_callback_t) (struct timer_struct *, void *); 
/* 定时器 */
typedef struct timer_struct {
    list_t list;                /* 定时器链表 */
    clock_t timeout;            /* 超时点，以ticks为单位 */
    void *arg;                  /* 参数 */
    unsigned long id;           /* 定时器的id值 */
    timer_callback_t callback;  /* 回调函数 */
} timer_t;
```

定时器有一个回调函数，就是超时时会去调用的函数，来执行使用者需要执行的内容。

## 墙上时间管理

墙上时间在xbook2中就是计算年月日，时分秒的时间。

```c
typedef struct {
	int second;         /* [0-59] */
	int minute;         /* [0-59] */
	int hour;           /* [0-23] */
	int day;            /* [1-31] */
	int month;          /* [1-12] */
	int year;           /* year */
	int week_day;       /* [0-6] */
	int year_day;       /* [0-366] */
} walltime_t;
```

最开始这些时间值是从硬件获取，获取后，我们便可以通过时钟节拍sys_ticks来判断是否更新这些时间值。

```c
void walltime_init()
{
    /* 初始化系统时间 */
    //用一个循环让秒相等
	do{
		walltime.year      = time_get_year();
		walltime.month     = time_get_month();
		walltime.day       = time_get_day();
		walltime.hour      = time_get_hour();
		walltime.minute    =  time_get_minute();
		walltime.second    = time_get_second();
		
		/* 转换成本地时间 */
		/* 自动转换时区 */
#ifdef CONFIG_TIMEZONE_AUTO
        if(walltime.hour >= 16){
			walltime.hour -= 16;
		}else{
			walltime.hour += 8;
		}
#endif /* CONFIG_TIMEZONE_AUTO */
	}while(walltime.second != time_get_second());

    walltime.week_day = walltime_get_week_day(walltime.year, walltime.month, walltime.day);
    walltime.year_day = walltime_get_year_days();

    walltime_printf();
}
```

```c
static void timer_softirq_handler(softirq_action_t *action)
{
    if (systicks % HZ == 0) {  /* 1s更新一次 */
        walltime_update_second();
    }
    timer_update_ticks();
    alarm_update_ticks();
}
```

用户程序可以通过系统调用来获取这个时间值，从而得知当前的日期和时间。

```c
int sys_get_walltime(walltime_t *wt)
{
    walltime_t tmp = walltime;
    --tmp.month;
    return mem_copy_to_user(wt, &tmp, sizeof(walltime_t));
}
```

