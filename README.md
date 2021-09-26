# Note
杂记

这里记录一些编程过程中的随想以及趣事






## Lua

### 为什么在早期的库里面 在luaL_openlibs之前要关闭gc
问题：  
最开始是在别人的代码中发现的这段逻辑  
这段gc看起来很高深莫测，似乎看不出道理来  
```
  lua_gc(L, LUA_GCSTOP, 0);  /* stop collector during initialization */
  luaL_openlibs(L);  /* open libraries */
  lua_gc(L, LUA_GCRESTART, 0);
```

原理：  

在搜索中发现最早的[lua5.1版本源码](https://github.com/lua/lua/blob/69ea087dff1daba25a2000dfb8f1883c17545b7a/lua.c#L332)中的就存在这段写法，
我们看到在luaL_openlibs之前强行关闭了gc，并在openlibs结束后开启  

关于这么做的用意，Roberto回答是[to reduce the GC overhead when creating large number of
objects that are not garbage](http://lua-users.org/lists/lua-l/2008-07/msg00690.html)
即在创建大量不需要垃圾回收对象时减少额外开销



结论：  
5.1和5.2是需要关掉gc的  
5.3开始已经没有类似的写法了

我们查看了我们正在使用的最新的lua5.4版本的源码，发现已经没有再使用类似的调用了，因此我们就不再沿用老的写法了

