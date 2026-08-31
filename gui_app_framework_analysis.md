# SiFli SDK GUI APP 调度与界面切换分析

当前 SDK 的 GUI APP 框架，本质上是“运行在 LVGL 主线程中的串行导航状态机”，不是给每个 APP 创建独立线程。它在 LVGL Screen 之上实现了两级管理：

- APP 级：类似最近使用列表，负责 APP 启动、前后台、淘汰、恢复。
- Page 级：每个 APP 内部维护页面栈，负责页面前进、返回、销毁。
- 所有操作都先进入 RT-Thread 邮箱，再由 LVGL 定时器统一执行，因此生命周期回调不会并发执行。

以下分析基于当前 `main` 分支提交 `a53233a6`。

## 1. 整体调用链

```text
gui_app_run / gui_app_goback / gui_app_create_page
                ↓
        10 项 RT-Thread mailbox
                ↓
     LVGL timer：每两个刷新周期执行一次
                ↓
          APP 状态机
                ↓
          Page 状态机
                ↓
      lv_scr_load 切换当前 Screen
                ↓
        LVSF 页面切换动画
```

框架初始化时创建邮箱和 LVGL Timer，没有新建调度线程；Timer 最终仍由 GUI 线程中的 `lv_timer_handler()` 驱动。[gui_app_fwk.c](middleware/app_fwk/gui_app_fwk.c#L950) [watch_demo.c](example/multimedia/lvgl/watch_v9/src/gui_apps/watch_demo.c#L465)

所以这里的“APP 调度”是界面和生命周期调度，不是 RTOS CPU 调度。

## 2. APP 与页面的数据结构

APP 由 `gui_runing_app_t` 表示，主要保存：

- APP ID、启动 Intent、入口函数。
- APP 自己的页面链表。
- 当前状态、目标状态。
- 在 running/suspend 链表中的节点。

页面由 `subpage_node_t` 表示，保存：

- 页面 ID。
- 独立的 LVGL Screen。
- 生命周期回调。
- enter/exit 动画参数。
- user data 和可选的页面私有内存。

定义集中在 [gui_app_int.h](middleware/app_fwk/gui_app_int.h#L62)。

运行中的 APP 按最近激活顺序排列：

```text
running_app_list → 当前 APP → 上一个 APP → 更早的 APP
```

每个 APP 的页面同样是栈式排列：

```text
page_list → 当前页面 → 上一个页面 → 更早页面
```

对应实现见 [app_schedule.c](middleware/app_fwk/app_schedule.c#L51)。

## 3. 生命周期状态机

页面生命周期是框架最核心的部分：

```text
CREATED
  │ ONSTART
  ▼
STARTED
  │ ONRESUME
  ▼
RESUMED
  │ ONPAUSE
  ▼
PAUSED
  │ ONSTOP
  ▼
STOPPED
```

`PAUSED` 可以重新经过 `ONRESUME` 回到前台。[app_schedule.c](middleware/app_fwk/app_schedule.c#L570)

各阶段的准确语义：

| 回调 | 框架行为 | APP 应做的工作 |
|---|---|---|
| ONSTART | 创建新 LVGL Screen，临时加载该 Screen | 创建控件、分配页面级数据 |
| ONRESUME | `lv_scr_load(page->scr)`，页面成为当前界面 | 启动 Timer、动画、订阅实时数据 |
| ONPAUSE | Screen 保留，但不再显示 | 停止 Timer、绘制任务、硬件访问 |
| ONSTOP | 回调结束后删除整个 Screen 并释放页面节点 | 释放非 LVGL 内存和外部资源 |

实际执行逻辑见 [app_schedule.c](middleware/app_fwk/app_schedule.c#L1641)。

一个重要结论：框架不会自动暂停 APP 自己创建的 LVGL Timer。后台页面的 Timer 仍可能继续运行，所以必须在 `ONPAUSE` 停止，在 `ONRESUME` 恢复。`rotation3d` 示例就是这样处理的。[rotation3d.c](example/multimedia/lvgl/watch_v9/src/gui_apps/rotation3d/rotation3d.c#L326)

## 4. 启动一个 APP 的完整过程

调用：

```c
gui_app_run("clock active=2");
```

流程是：

1. 将字符串解析成固定 128 字节的 Intent，首项是 APP ID，其余为参数。
2. 把 `GUI_APP_MSG_RUN_APP` 投递进邮箱。
3. 调度器按 ID 查找已经运行的 APP。
4. 未运行时，从内建 APP、脚本 APP、工具 APP或动态模块中解析入口函数。
5. 调用 APP 的 `app_main(intent)`。
6. APP 入口注册 root 页面。
7. root 页面依次执行 `ONSTART → ONRESUME`。
8. 原前台页面收到 `ONPAUSE`。
9. 启动界面切换动画。

入口代码在 [gui_app_fwk.c](middleware/app_fwk/gui_app_fwk.c#L231)，APP 加载和启动在 [app_schedule.c](middleware/app_fwk/app_schedule.c#L987)。

同一个 APP 再次启动时：

- Intent 完全相同：直接恢复已有 APP，不重复执行 APP 入口。
- Intent 不同：销毁旧实例的所有页面，然后重新执行入口。
- APP 因数量限制进入 suspend 链表：重新执行入口、重新创建页面。

因此 Intent 参数变化实际上具有“重启 APP”的效果。

## 5. APP 注册方式

当前代码保留了两套注册方式。

底层方式：

```c
static int app_main(intent_t i)
{
    gui_app_regist_msg_handler(APP_ID, msg_handler);
    return 0;
}

BUILTIN_APP_EXPORT(..., APP_ID, app_main, 1);
```

watch 示例使用这种方式。[app_mainmenu.c](example/multimedia/lvgl/watch_v9/src/gui_apps/main/app_mainmenu.c#L1534)

封装方式：

```c
APPLICATION_REGISTER(...);
APP_PAGE_REGISTER(app_id, page_id, private_mem_size);
```

它会把 APP 描述和子页面描述放进链接段，`gui_app_run_subpage()` 可以按 APP ID/Page ID 查找。[app_reg.h](middleware/app_fwk/reg_fwk/app_reg.h#L140)

`gui_app_init(style)` 中的 `style` 用于选择 `BuiltinApp1Tab`、`BuiltinApp2Tab` 等应用表，而不是 LVGL 版本。

## 6. 同一 APP 内的页面切换

打开新页面：

```c
gui_app_create_page("detail", detail_msg_handler);
```

框架会：

1. 分配页面节点和可选私有内存。
2. 将当前页面目标状态设为 `PAUSED`。
3. 将新页面目标状态设为 `RESUMED`。
4. 新页面执行 `ONSTART → ONRESUME`。
5. 新页面移动到页面链表头部。

实现见 [app_schedule.c](middleware/app_fwk/app_schedule.c#L1217)。

返回：

```c
gui_app_goback();
```

行为是：

- 当前 APP 有多个页面：当前页面 `ONPAUSE → ONSTOP`，前一个页面 `ONRESUME`。
- 当前 APP 只有一个页面：退出整个 APP，并恢复前一个 APP。
- 已是系统最后一个页面：恢复 suspend 历史 APP；没有历史则启动 `Main`。

`gui_app_goback_to_page(id)` 会销毁目标页面上面的所有页面，类似 pop-to。[app_schedule.c](middleware/app_fwk/app_schedule.c#L1189)

为了避免删除仍是当前 Screen 的对象，框架会先加载恢复页面，再对退出页面执行 `ONSTOP` 和 `lv_obj_del(screen)`。

## 7. APP 之间切换和数量限制

默认最多保留两个 running APP：

```text
当前 APP + 一个后台 APP
```

配置来自 [Kconfig](middleware/app_fwk/Kconfig#L18)。

启动第三个 APP 时，最老的 APP会：

- 所有页面执行 `ONSTOP`。
- Screen 和页面节点被释放。
- APP 节点进入 `suspend_app_list`。
- 返回时重新执行 APP 入口，重建页面。

所以框架中的 “suspended APP” 不是保留完整 UI 的后台 APP；它只保留 APP 身份、Intent、入口等轻量信息。

`Main` 是特殊 APP：恢复 `Main` 时会销毁其他 running 和 suspended APP，相当于清理导航历史。[app_schedule.c](middleware/app_fwk/app_schedule.c#L741)

因此：

```c
gui_app_run("Main");
```

不是普通返回，而更接近“回桌面并清栈”。

## 8. 界面切换动画

每个页面分别保存 enter 和 exit 动画参数。切换时框架同时拿到：

- 即将进入页面的 `a_enter`
- 即将退出页面的 `a_exit`

然后根据优先级决定最终动画类型。[lvsf_switchanim_com.c](middleware/lvgl/lvsf_v9/lvsf_switchanim_com.c#L57)

默认行为：

- 动画类型：Push。
- 时长：300ms。
- 前进和返回方向由框架判断。
- 动画期间暂停 APP 调度。
- 如果邮箱中出现新操作，会中止当前动画，优先处理新操作。

Kconfig 提供：

- `APP_TRANS_ANIMATION_NONE`：无动画、无动画缓冲。
- `APP_TRANS_ANIMATION_OVERWRITE`：覆盖式、无额外缓冲。
- `APP_TRANS_ANIMATION_SCALE`：默认，使用两个截图缓冲。

截图内存约为：

```text
2 × 屏幕宽 × 屏幕高 × 色深 / 8
```

例如 RGB565 的两个全屏缓冲约占 `4 × width × height` 字节。

具体动画接入见 [gui_app_trans_anim.c](middleware/app_fwk/lvgl_v9/gui_app_trans_anim.c#L100)。

需要特别注意：LVGL v8 已接入滑动返回手势；LVGL v9 文件明确标记手势模块尚未实现，交互式手势动画当前被关闭，普通自动切换动画仍然可用。[v9 gui_app_trans_anim.c](middleware/app_fwk/lvgl_v9/gui_app_trans_anim.c#L13)

## 9. 当前 watch 示例的真实路径

启动阶段：

```text
littlevgl2rtt_init
→ resource_init
→ gui_app_init(1)
→ gui_app_run("Main")
→ lv_timer_handler 循环
```

主菜单图标点击后直接调用 `gui_app_run(cmd)`；ESC 键调用 `gui_app_goback()`。[watch_demo.c](example/multimedia/lvgl/watch_v9/src/gui_apps/watch_demo.c#L67) [app_mainmenu.c](example/multimedia/lvgl/watch_v9/src/gui_apps/main/app_mainmenu.c#L1186)

这套示例基本就是框架推荐的运行方式。

## 10. 开发中最需要注意的问题

- 所有 LVGL 控件必须在 GUI host 线程操作。虽然少数 API 没有显式线程断言，但消息发送路径也会直接操作 LVGL 输入设备，不能把它当成完全线程安全的跨线程接口。
- 生命周期回调必须短，不能阻塞；它们和 LVGL 刷新运行在同一个线程。
- 不要在 APP 内直接调用 `lv_scr_load()`，否则框架保存的 active APP/Page 与真实 Screen 会失配。
- `ONPAUSE` 不等于资源销毁；Screen 仍然存在。
- `ONSTOP` 返回后 Screen 会被框架删除，不要继续保存或访问其中的控件指针。
- 回调参数 `param` 实际上传入的是 APP ID，不是页面 user data。页面数据应通过 `gui_app_this_page_userdata()` 或 `gui_app_this_page_memory()` 获取。
- 邮箱容量只有 10，满时当前实现会触发断言；程序连续批量发送导航命令需要自行节流。
- APP ID 数组只有 16 字节，但启动路径存在直接 `strcpy`，外部输入必须限制为最多 15 个可见字符。
- LVGL v9 当前不要依赖交互式侧滑返回。

调试时可以使用 Finsh/MSH 命令：

```text
list_app
app_run <app> [params]
app_goback
app_exit <app>
app_sche_print_perf_tick 1
```

这些命令会打印 running/suspend APP、页面状态和生命周期耗时，定义在 [gui_app_fwk.c](middleware/app_fwk/gui_app_fwk.c#L1305)。
