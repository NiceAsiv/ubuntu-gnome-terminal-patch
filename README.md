# Terminal selection copy and right-click paste function

Based on personal-testing,this method is applicable to gnome-terminal versions 3.36 to 3.44 in ubuntu22.

**0x01.Get the gnome terminal source code**

```shell
sudo apt-get update
apt-get source gnome-terminal
```



**0x02.Install GNOME development environment**

```shell
sudo apt-get build-dep gnome-terminal
```

If any third-party libraries are missing, please install them.

the only file that needs to be modified is `gnome-terminal-3.44.0/src/terminal-screen.c`(the version of GNOME Terminal used by me)

**0x03.Define event functions**

Add a release event after the press event.

```c
static gboolean terminal_screen_popup_menu (GtkWidget *widget);
static gboolean terminal_screen_button_press (GtkWidget *widget,
                                              GdkEventButton *event);
// Add button release event
static gboolean terminal_screen_button_release (GtkWidget *widget,
                                              GdkEventButton *event);
static void terminal_screen_hierarchy_changed (GtkWidget *widget,
                                               GtkWidget *previous_toplevel);
```



**0x04.Bind the execution function for the button press and release events.**

add `widget_class->button_release_event = terminal_screen_button_release;` in fuction `static void
terminal_screen_class_init (TerminalScreenClass *klass)`

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
//Bind the execution function for button press event
  terminal_class->child_exited = terminal_screen_child_exited;
```



**0x05.Modify the logic of the right-click event execution function**

```c
static gboolean
terminal_screen_button_press (GtkWidget      *widget,
                              GdkEventButton *event)
{
  TerminalScreen *screen = TERMINAL_SCREEN (widget);
  gboolean (* button_press_event) (GtkWidget*, GdkEventButton*) =
    GTK_WIDGET_CLASS (terminal_screen_parent_class)->button_press_event;
  gs_free char *hyperlink = nullptr;
  gs_free char *url = nullptr;
  int url_flavor = 0;
  gs_free char *number_info = nullptr;
  gs_free char *timestamp_info = nullptr;
  guint state;

  state = event->state & gtk_accelerator_get_default_mod_mask ();

  hyperlink = terminal_screen_check_hyperlink (screen, (GdkEvent*)event);
  url = terminal_screen_check_match (screen, (GdkEvent*)event, &url_flavor);
  terminal_screen_check_extra (screen, (GdkEvent*)event, &number_info, &timestamp_info);

  if (hyperlink != nullptr &&
      (event->button == 1 || event->button == 2) &&
      (state & GDK_CONTROL_MASK))
    {
      gboolean handled = FALSE;

      g_signal_emit (screen, signals[MATCH_CLICKED], 0,
                     hyperlink,
                     FLAVOR_AS_IS,
                     state,
                     &handled);
      if (handled)
        return TRUE; /* don't do anything else such as select with the click */
    }

  if (url != nullptr &&
      (event->button == 1 || event->button == 2) &&
      (state & GDK_CONTROL_MASK))
    {
      gboolean handled = FALSE;

      g_signal_emit (screen, signals[MATCH_CLICKED], 0,
                     url,
                     url_flavor,
                     state,
                     &handled);
      if (handled)
        return TRUE; /* don't do anything else such as select with the click */
    }
//Modify here
  if (event->type == GDK_BUTTON_PRESS && event->button == 3)
    {
      if (!(event->state & (GDK_SHIFT_MASK | GDK_CONTROL_MASK | GDK_MOD1_MASK)))
        {
          /* on right-click, we should first try to send the mouse event to
           * the client, and popup only if that's not handled. */
          //if (button_press_event && button_press_event (widget, event))
           // return TRUE;
          terminal_screen_do_popup (screen, event, hyperlink, url, url_flavor, number_info, timestamp_info);
          // Comment out the above 3 lines of code logic, directly call the paste function
          //add this in version 3.44
          vte_terminal_paste_clipboard (VTE_TERMINAL (screen));
          
          hyperlink = nullptr; /* adopted to the popup info */
          url = nullptr; /* ditto */
          number_info = nullptr; /* ditto */
          timestamp_info = nullptr; /* ditto */
          return TRUE;
        }
      else if (!(event->state & (GDK_CONTROL_MASK | GDK_MOD1_MASK)))
        {
          /* do popup on shift+right-click */
          terminal_screen_do_popup (screen, event, hyperlink, url, url_flavor, number_info, timestamp_info);
          hyperlink = nullptr; /* adopted to the popup info */
          url = nullptr; /* ditto */
          number_info = nullptr; /* ditto */
          timestamp_info = nullptr; /* ditto */
          return TRUE;
        }
    }
```



**0x06.Implement the selection copy function**

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

**0x07.Compile and install**

After making the necessary changes in the source code, go to the `gnome-terminal-3.44.0` directory and execute:

```shell
dpkg-buildpackage -us -uc -b
```

install

```
sudo dpkg -i gnome-terminal_3.44.0-1ubuntu1_amd64.deb
```

![image-20231028222659981](https://cdn.niceasiv.cn/image-20231028222659981.png)



Restart the terminal, and you will be able to get Terminal selection copy and right-click paste function
