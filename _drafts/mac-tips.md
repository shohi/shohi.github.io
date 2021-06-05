---
layout: post
title: Mac tips
category: tech
guid: 519E87D6-30D0-418B-B29A-C5872305C5BD
tags: [tech, mac]

---
{% include JB/setup %}

## Keyboard

[Karabiner-Elements](https://pqrs.org/osx/karabiner/)是一款开源且十分优秀的键盘修改器, 允许自定义键盘映射.

```
# install
brew cask install karabiner-elements
```

- 当接上外部键盘时, 禁用内置键盘

- ikbc-c87键盘说明书

```
Fn + F9     => 静音
Fn + F10    => 音量减
Fn + F11    => 音量加

Fn + F12    => 锁定双Win
Fn + PrtSc  => 解锁双Win

Fn + ScrLK  => 六键无冲/全键无冲 切换

Fn + Delete => 恢复出厂设置, 长按3秒
```

尤其需要注意`锁定双Win`, 将导致`karabiner`的映射失效.


## Application

- 使用键盘[打开最小化应用](https://zhuanlan.zhihu.com/p/25746402)

```
1. 按下<Commad>, 一直按着
2. 连续按<TAB>, 找到你要打开的应用类型, 之后松开
3. 使用上下方向键调出所有的实例窗口, 此时<Command>可以松开
4. 使用方向键选择要打开的窗口, 然后按下Enter即可
```

问题: 如何将上述1-3步操作映射到一个快捷键上, e.g. `fn` + 向下箭头?
