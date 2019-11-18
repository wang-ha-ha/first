##### 相关手册

- GUIX_User_Guide.pdf    				 GUIX介绍、还有所有API的介绍
- GUIX_Studio_User_Guide.pdf       GUIX Studio 软件使用介绍
- synergy-ssp-v163.pdf                     瑞萨软件框架介绍

##### GUIX 

- 创建工程时要注意选择CPU等，在瑞萨平台要选瑞萨的，使用VS选择通用，还有编译环境，瑞萨要选择GUN

##### 屏幕配置

- 3.4寸屏要注意SPI引脚的配置，需要选择一个SPI框架，16根数据输出线
  - LCD_RESET、LCD_CMD、LCD_CS 引脚的配置，并在lcd.h 里面修改
- 7寸屏没有SPI的配置，24根数据输出线
  - 7寸屏不需要 lcd.h 和 lcd_setup.c这两个文件
  - 7寸屏使用片内RAM内存不够，800 * 480 * 2 （一个像素点2个字节） = 768，000  大概是700多Kb，双缓存就是1.4M多，片内RAM是640K；所有需要添加外部RAM
  - 需要在XML添加SDRAM数据地址总线 
  - 因为我创建工程时选择版型为客户，所有需要自己添加SDARM的初始化函数。我直接把以前工程生成的移植过来
- 根据数据输出线选择输出的RGB位数
- 根据屏幕尺寸选择输出大小
- 生成一个GUIX on gx的框架
- Hsync、Vsync、DataEnable 引脚的配置
- 

##### 屏幕初始化流程

- Windows模拟器

  ```c
  /* Initialize GUIX.  */
  gx_system_initialize();
  
  gx_studio_display_configure(DISPLAY_1, win32_graphics_driver_setup_24xrgb, \
  LANGUAGE_CHINESE, DISPLAY_1_THEME_1, &root);
  
  /* create the thermometer screen */
  gx_studio_named_widget_create("splash_screen", (GX_WIDGET *) root, NULL);
  
  /* Show the root window to make it and patients screen visible.  */
  gx_widget_show(root);
  
  /* let GUIX thread run */
  gx_system_start();
  ```
  
- 瑞萨平台（synergy-ssp-v163.pdf  1212页）

  - 先在XML 中添加GUIX on gx
  - 配置好输出格式
  - 配置引脚 GLCD
  
  ```c
  /* Initializes GUIX. */
  status = gx_system_initialize();
  if(TX_SUCCESS != status)
  {
      while(1);
  }
  
  /* Initializes GUIX drivers. */
  err = g_sf_el_gx.p_api->open (g_sf_el_gx.p_ctrl, g_sf_el_gx.p_cfg);
  if(SSP_SUCCESS != err)
  {
      while(1);
  }
  
  gx_studio_display_configure ( DISPLAY_1,
                                    g_sf_el_gx.p_api->setup,
                                    LANGUAGE_ENGLISH,
                                    DISPLAY_1_THEME_1,
                                    &p_window_root );
  
  err = g_sf_el_gx.p_api->canvasInit(g_sf_el_gx.p_ctrl, p_window_root);
  if(SSP_SUCCESS != err)
  {
      while(1);
  }
  
  /* Creates the primary screen. */
  status = gx_studio_named_widget_create("splash_screen",
                                         (GX_WIDGET *)p_window_root, GX_NULL);
  if(TX_SUCCESS != status)
  {
      while(1)
      {;}
  }
  
  /* Shows the root window to make it and patients screen visible. */
  status = gx_widget_show(p_window_root);
  if(TX_SUCCESS != status)
  {
      while(1)
      {;}
  }
  
  /* Lets GUIX run. */
  status = gx_system_start();
  
  /* Controls the GPIO pin for LCD ON. */
  g_ioport.p_api->pinWrite(IOPORT_PORT_03_PIN_13, IOPORT_LEVEL_HIGH);
  
  /* Controls the GPIO pin for BL ON. */
  R_BSP_SoftwareDelay(200,BSP_DELAY_UNITS_MILLISECONDS);
  g_ioport.p_api->pinWrite((ioport_port_pin_t)IO_PORT_LCD_BACKLIGHT_ON, IOPORT_LEVEL_HIGH);
  
  /* Opens PWM driver and controls the TFT panel back light. */
  err = g_pwm_backlight.p_api->open(g_pwm_backlight.p_ctrl, g_pwm_backlight.p_cfg);
  if (err)
  {
      while(1)
    {;}
  }
  
  ```

##### 触摸配置

- IIC通道选择，芯片地址确认
  - 7寸屏的从机地址为0x38，iic通道2
  - 3.4寸屏
- 触摸芯片中断引脚、复位引脚的配置
- 触摸框架会使用一个消息队列框架 messaging framework 注意在 XML Messaging 里面Touch消息要给使用触摸框架的线程订阅

##### 触摸处理流程

```c
{
    	g_sf_message.p_api->pend(g_sf_message.p_ctrl, &hmiThread_message_queue, 	、
                          \(sf_message_header_t **) &p_message, TX_WAIT_FOREVER);
        switch (p_message->event_b.class_code)
        {
        case SF_MESSAGE_EVENT_CLASS_TOUCH:
            if (SF_MESSAGE_EVENT_NEW_DATA == p_message->event_b.code)
            {
                sf_touch_panel_payload_t * p_touch_message =
                    (sf_touch_panel_payload_t *) p_message;

                /** Translate a touch event into a GUIX event */
                hmi_send_touch_message(p_touch_message);
            }
            break;
        default:
            break;
        }

        /** Message is processed, so release buffer. */
        err = g_sf_message.p_api->bufferRelease(g_sf_message.p_ctrl, 
                \(sf_message_header_t *) p_message, SF_MESSAGE_RELEASE_OPTION_NONE);
        if (err)
        {
            while(1)
            {;}
        }
}
static void hmi_send_touch_message(sf_touch_panel_payload_t * p_payload)
{
    bool send_event = true;
    GX_EVENT gxe;

    switch (p_payload->event_type)
    {
    case SF_TOUCH_PANEL_EVENT_DOWN:
        gxe.gx_event_type = GX_EVENT_PEN_DOWN;
        break;
    case SF_TOUCH_PANEL_EVENT_UP:
        gxe.gx_event_type = GX_EVENT_PEN_UP;
        break;
    case SF_TOUCH_PANEL_EVENT_HOLD:
    case SF_TOUCH_PANEL_EVENT_MOVE:
        gxe.gx_event_type = GX_EVENT_PEN_DRAG;
        break;
    case SF_TOUCH_PANEL_EVENT_INVALID:
        send_event = false;
        break;
    default:
        break;
    }

    if (send_event)
    {
        /** Send event to GUI */
        gxe.gx_event_sender         = GX_ID_NONE;
        gxe.gx_event_target         = 0;  /* the event to be routed to the widget that has input focus */
        gxe.gx_event_display_handle = 0;

        gxe.gx_event_payload.gx_event_pointdata.gx_point_x = p_payload->x;
#if defined(BSP_BOARD_S7G2_SK)
        gxe.gx_event_payload.gx_event_pointdata.gx_point_y = (GX_VALUE)(320 - p_payload->y);
#else
        gxe.gx_event_payload.gx_event_pointdata.gx_point_y = p_payload->y;
#endif

        gx_system_event_send(&gxe);
    }
}
```



##### 事件处理流程

- 在GUIX Studio 工具中Properties View下页面有个Event Function 属性，这个属性就是页面事件处理回调函数的名称
- GUIX_User_Guide.pdf 44 有对控件事件的介绍
- 每个事件都有自己默认的事件处理函数。gx_ < widget-type > _event_process
- 控件都可以去修改Event Function去改写事件事件处理，gx_widget_event_generate 使用这个发送事件到父控件

 ```c
//example
UINT SplashScreenEventHandler(GX_WINDOW *widget, GX_EVENT *event_ptr)
{
    UINT status = 0;
    switch(event_ptr->gx_event_type)
    {
            //调用gx_widget_show 时发送进来的事件 
        case GX_EVENT_SHOW:
            //把GUIX的定时器指定为当前控件 
            gx_system_timer_start(widget, TIME_EVENT_TIMER, 100 * 3, 0);
            return gx_window_event_process(widget, event_ptr);
            break;
            //定时器时间到了发送的事件
        case GX_EVENT_TIMER:
            {            
                /** create the product screen */
                gx_studio_named_widget_create("product_screen", GX_NULL, GX_NULL);

                gx_widget_delete(widget);
                if (root->gx_widget_first_child == GX_NULL)
                {
                    gx_widget_attach(root, &product_screen);
                }
                break;
            }
        /*
        * 子控件的事件 
  		* ID_LOGO_SPRITE，GX_EVENT_SPRITE_COMPLETE为控件ID
  		*/
        case GX_SIGNAL(ID_LOGO_SPRITE, GX_EVENT_SPRITE_COMPLETE):
            gx_widget_hide(&splash_screen.splash_screen_sprite);
            MoveSprite();
            //gx_sprite_start(&splash_screen.splash_screen_sprite,0);
            gx_widget_show(&splash_screen.splash_screen_sprite);
            break;
        //GX_EVENT_CLICKED 为点击事件
        case GX_SIGNAL(IDB_CHILD_BUTTON, GX_EVENT_CLICKED):
            /* insert your button handler code here */
            break;
        default:
            status = gx_window_event_process(widget, event_ptr);
            break;

    }
    return status;
}
 ```

##### 事件类型

```c
/* gx_api.h 467 */
/* Define the pre-defined Widget event types.  */

#define GX_EVENT_TERMINATE                  1
#define GX_EVENT_REDRAW                     2
#define GX_EVENT_SHOW                       3
#define GX_EVENT_HIDE                       4
#define GX_EVENT_RESIZE                     5
#define GX_EVENT_SLIDE                      6
#define GX_EVENT_FOCUS_GAINED               7
#define GX_EVENT_FOCUS_LOST                 8
#define GX_EVENT_HORIZONTAL_SCROLL          9
#define GX_EVENT_VERTICAL_SCROLL            10
#define GX_EVENT_TIMER                      11
#define GX_EVENT_PEN_DOWN                   12
#define GX_EVENT_PEN_UP                     13
#define GX_EVENT_PEN_DRAG                   14
#define GX_EVENT_KEY_DOWN                   15
#define GX_EVENT_KEY_UP                     16
#define GX_EVENT_CLOSE                      17
#define GX_EVENT_DESTROY                    18
#define GX_EVENT_SLIDER_VALUE               19
#define GX_EVENT_TOGGLE_ON                  20
#define GX_EVENT_TOGGLE_OFF                 21
#define GX_EVENT_RADIO_SELECT               22
#define GX_EVENT_RADIO_DESELECT             23
#define GX_EVENT_CLICKED                    24
#define GX_EVENT_LIST_SELECT                25
#define GX_EVENT_VERTICAL_FLICK             26
#define GX_EVENT_HORIZONTAL_FLICK           28
#define GX_EVENT_MOVE                       29
#define GX_EVENT_PARENT_SIZED               30
#define GX_EVENT_CLOSE_POPUP                31
#define GX_EVENT_ZOOM_IN                    32
#define GX_EVENT_ZOOM_OUT                   33
#define GX_EVENT_LANGUAGE_CHANGE            34
#define GX_EVENT_RESOURCE_CHANGE            35
#define GX_EVENT_ANIMATION_COMPLETE         36
#define GX_EVENT_SPRITE_COMPLETE            37
#define GX_EVENT_TEXT_EDITED                40
#define GX_EVENT_TX_TIMER                   41
#define GX_EVENT_FOCUS_NEXT                 42
#define GX_EVENT_FOCUS_PREVIOUS             43
#define GX_EVENT_FOCUS_GAIN_NOTIFY          44
#define GX_EVENT_SELECT                     45
#define GX_EVENT_DESELECT                   46
#define GX_EVENT_PROGRESS_VALUE             47
#define GX_EVENT_TOUCH_CALIBRATION_COMPLETE 48
#define GX_EVENT_INPUT_RELEASE              49
```



##### 页面切换

```c
/** create the product screen */
gx_studio_named_widget_create("product_screen", GX_NULL, GX_NULL);
//删除当前页面
gx_widget_delete(widget);
if (root->gx_widget_first_child == GX_NULL)
{
    //把需要跳转的页面添加根页面
    gx_widget_attach(root, &product_screen);
}
```

##### GUIX API

###### gx_system_timer_start

 ```c
//GUIX_User_Guide.pdf 496
UINT gx_system_timer_start(GX_WIDGET *owner, UINT timer_id,
                             UINT initial_ticks,
  						   UINT reschedule_ticks);
Description:
  	This service starts a timer for the specified widget.
Parameters：
    owner                          Pointer to widget control block
    timer_id                       ID of timer
    initial_ticks                  Initial expiration ticks
    reschedule_ticks               Periodic expiration ticks        
  
 ```

######   gx_widget_event_generate 

```c
//GUIX_User_Guide.pdf 602
UINT gx_widget_event_generate(GX_WIDGET *widget, USHORT
								event_type, LONG value);
Parameters:
    widget 				Pointer to widget
    event_type 			Type of event. Appendix E contains pre-
    defined 			GUIX events. Additional eventsmay be added by the application.
    value 				Additional event information

```

###### gx_widget_attach

```c
//GUIX_User_Guide.pdf 507
UINT gx_widget_attach(GX_WIDGET *parent, GX_WIDGET *widget);

Description:
	Attach widget to parent

Parameters：
	parent 			Pointer to parent widget
	widget			Pointer to child widget
```

##### 问题

- 点3.4寸屏的时候刷图有问题