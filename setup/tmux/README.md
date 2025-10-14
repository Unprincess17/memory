```
# 把时间设置成HH:MM:SS MM-DD格式
tmux set-option -g status-right  "#{?window_bigger,[#{window_offset_x}#,#{window_offset_y}] ,}\"#{=21:pane_title}\" %H:%M:%S %b-%d"
```