sublime 在 mac 上的 home/end 键是直接返回到文档开头/结尾的

要想改为返回到本行开头/结尾，进入 `Preferences > Key Bindings - User`，在右侧粘贴以下配置：

```json
[
	{ "keys": ["home"], "command": "move_to", "args": {"to": "bol"} },
	{ "keys": ["end"], "command": "move_to", "args": {"to": "eol"} },
	{ "keys": ["shift+end"], "command": "move_to", "args": {"to": "eol", "extend": true} },
	{ "keys": ["shift+home"], "command": "move_to", "args": {"to": "bol", "extend": true } },
	{ "keys": ["ctrl+home"], "command": "move_to", "args": {"to": "bof"} },
	{ "keys": ["ctrl+end"], "command": "move_to", "args": {"to": "eof"} }
]
```

