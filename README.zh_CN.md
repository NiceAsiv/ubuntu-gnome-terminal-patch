#gnome终端实现选中复制和右键粘贴功能

基于个人测试，该方法适用于 gnome-terminal 版本 3.36 到 3.44。

**0x01. 获取 gnome 终端源码**

```shell
sudo apt-get update
apt-get source gnome-terminal
```

**0x02. 安装 GNOME 开发环境**

```shell
sudo apt-get build-dep gnome-terminal
```

如果缺失第三方库，请手动安装。

唯一需要修改的文件是 `gnome-terminal-3.44.0/src/terminal-screen.c`（我所使用的 GNOME 终端版本）。

**0x03. 定义事件函数**

在按压事件后增加释放事件。

```c
static gboolean terminal_screen_popup_menu (GtkWidget *widget);
static gboolean terminal_screen_button_press (GtkWidget *widget,
                                              GdkEventButton *event);
// 增加按压释放事件
static gboolean terminal_screen_button_release (GtkWidget *widget,
                                              GdkEventButton *event);
static void terminal_screen_hierarchy_changed (GtkWidget *widget,
                                               GtkWidget *previous_toplevel);
```

**0x04. 绑定按压释放事件的执行函数**

在函数 `static void terminal_screen_class_init (TerminalScreenClass *klass)` 中添加 `widget_class->button_release_event = terminal_screen_button_release;`。

```c
terminal_screen_class_init (TerminalScreenClass *klass)
{
  GObjectClass *object_class = G_OBJECT_CLASS (klass);
  GtkWidgetClass *widget_class = GTK_WIDGET_CLASS(klass);
  VteTerminalClass *terminal_class = VTE_TERMINAL_CLASS (klass);
  GSettings *settings;

  object_class->constructed = terminal_screen_constructed;
  object_class->dispose = terminal_screen_dispose;
  object_class->finalize = terminal_screen_finalize;
  object_class->get_property = terminal_screen_get_property;
  object_class->set_property = terminal_screen_set_property;

  widget_class->realize = terminal_screen_realize;
  widget_class->style_updated = terminal_screen_style_updated;
  widget_class->drag_data_received = terminal_screen_drag_data_received;
  widget_class->button_press_event = terminal_screen_button_press;
  widget_class->button_release_event = terminal_screen_button_release;
  widget_class->popup_menu = terminal_screen_popup_menu;
  widget_class->hierarchy_changed = terminal_screen_hierarchy_changed;
  // 绑定按压释放事件的执行函数
  terminal_class->child_exited = terminal_screen_child_exited;
```

**0x05. 修改右键事件的执行函数逻辑**

```c
static gboolean
terminal_screen_button_press (GtkWidget      *widget,
                              GdkEventButton *event)
{
  // ... 代码略

  if (event->type == GDK_BUTTON_PRESS && event->button == 3)
    {
      if (!(event->state & (GDK_SHIFT_MASK | GDK_CONTROL_MASK | GDK_MOD1_MASK)))
        {
          // ... 代码略
          // 添加这一行（在3.44版本）
          vte_terminal_paste_clipboard (VTE_TERMINAL (screen));
          
          hyperlink = nullptr; /* adopted to the popup info */
          url = nullptr; /* ditto */
          number_info = nullptr; /* ditto */
          timestamp_info = nullptr; /* ditto */
          return TRUE;
        }
      else if (!(event->state & (GDK_CONTROL_MASK | GDK_MOD1_MASK)))
        {
          // ... 代码略
        }
    }
```

**0x06. 实现选中复制函数**

```c
static gboolean
terminal_screen_button_release (GtkWidget *widget,
                                GdkEventButton *event){
    gboolean ret;
 
    TerminalScreen *screen = TERMINAL_SCREEN (widget);
    gboolean (* button_release_event) (GtkWidget*, GdkEventButton*) =
            GTK_WIDGET_CLASS (terminal_screen_parent_class)->button_release_event;
    ret = FALSE;
    if (button_release_event){
        ret = button_release_event (widget, event);
    }
 
    if (event->button == 1){
        gboolean can_copy;
        can_copy = vte_terminal_get_has_selection (VTE_TERMINAL (screen));
 
        if (can_copy)
            vte_terminal_copy_clipboard (VTE_TERMINAL (screen));
    }
 
    return ret;
}
```

**0x07. 编译和安装**

在源代码中进行必要的更改后，进入 `gnome-terminal-3.44.0` 目录并执行：

```shell
dpkg-buildpackage -us -uc -b
```

安装：

```shell
sudo dpkg -i gnome-terminal_3.44.0-1ubuntu1_amd64.deb
```

重新启动终端，你将能够使用终端选中复制和右键粘贴功能。
