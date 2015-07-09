title: lua中的_ENV
categories: 语言
tags: lua
---
在lua中，至少从5.2开始就不再有全局变量:
>当你写a = 1的时候，其实被编译成 _ENV.a = 1

但是一直以为_ENV是全局environment，其实不然，lua手册中说：
>every chunk is compiled in the scope of an external local variable called _ENV (see §3.3.2), so _ENV itself is never a global name in a chunk.

这里的external local variable就是upvalue，也就是_ENV是当前chunk的upvalue。

>When Lua compiles a chunk, it initializes the value of its _ENV upvalue with the global environment (see load).

当lua编译一个chunk的时候，如果不指定的话，默认使用全局environment初始化它的upvalue _ENV（其实就是引用），它是隐式声明的一个upvalue，看load声明：

>load (ld [, source [, mode [, env］])
loadfile ([filename [, mode [, env］])

所以_ENV是语法糖，它在lua编译的作为upvalue被隐式声明，并且可以由外部指定。
通过下面代码可以验证：

######t.lua
```
local _env = {tostring=tostring,print=print, debug=debug} 
print ("_ENV = "..tostring(_ENV))                         
print ("_env = "..tostring(_env))                         
local f, err = loadfile("m.lua", "bt", _env)              
assert(f, err)                                            
local m = f(f)                                            
m.test()                                                  
```
######m.lua
```
local f = ...                              
local m = {}                               
                                           
function m.test()                          
    local i =1                             
    while true do                          
        local n, v = debug.getupvalue(f, i)
        if not n then                      
            break                                                        
        elseif n == "_ENV" then                
            print ("_ENV = "..tostring(v)) 
            break                          
        end                                
        i = i+1                            
    end                                    
end                                        
                                           
return m                               
```
######输出
```
_ENV = table: 0x132f5f0
_env  = table: 0x1335f90
_ENV = table: 0x1335f90
```
查看指令码可以看到SETUPVAL：

######t.lua
```
_ENV = nil 
```
```
$ luac -l t.lua
main <t.lua:0,0> (3 instructions at 0x245b3c0)
0+ params, 2 slots, 1 upvalue, 0 locals, 0 constants, 0 functions
        1       [47]    LOADNIL         0 0
        2       [47]    SETUPVAL      0 0     ; _ENV
        3       [47]    RETURN          0 1
```
ps:
全局environment在C Registry(LUA_REGISTRYINDEX):LUA_RIDX_GLOBALS: At this index the registry has the global environment.
_G在初始化的时候被复制为全局environment。
