##### 按单词前进、后退、删除

进入 *Preferences > Profiles > Keys > General*，先将 `Left option` 勾选为 `+Esc`。再新增 3 个快捷键：



##### 按单词删除
不需要配置，使用时直接 `⌥+删除` 即可

如果无效，则配置如下：

```go
Keyboard Shortcut:⌥←Delete
Action: Send Hex Code
0x1B 0x08
```



##### 删除整行

```
Keyboard Shortcut: ⌘←Delete
Action: Send Hex Code
0x15
```



##### 按单词后退
```
新增快捷键 ⌥+←
Action 选择 Send Escape Sequence
Esc+ 填入 b
```



##### 按单词前进
```
新增快捷键⌥+→
Action 选择 Send Escape Sequence
Esc+ 填入 f
```



#### 参考
https://gcollazo.com/making-iterm-2-work-with-normal-mac-osx-keyboard-shortcuts/
