# How to add anchor in JavaScript?

> Refer to [d0c_s4vage's blog](http://d0cs4vage.blogspot.hk/2013/06/windbg-tricks-javascript-windbg.html)

A better way to instrument windbg via javascript is to create a way for javascript to print a message in windbg (and trigger a break): 
```
bu jscript9!Js::Math::Atan ".printf \"DEBUG: %mu\\n\", poi(poi(esp+10)+c);g"
bu jscript9!Js::Math::Atan2 ".printf \"DEBUG: %mu\\n\", poi(poi(esp+14)+c)"
```
(If you want to break, remove the ; g)

That's cool, but what if you want to do something a little more complicated, like track all allocations of a specific size after certain javascript statements have been executed. With the previous method, the javascript would have to look something like this:
```
function debug_stop(text){
    Math.atan2(0xbadc0de, text);
}

function debug(text){
    Math.atan(text);
}
```


# How to add anchor in Flash?

By modifying ActionScript code in Flash and recompile a new flash, we could use this following method.
> Refer to [PEDIY blog](https://bbs.pediy.com/thread-193313.htm)

In ActionScript, adding following **ExternalInterface.call** to anywhere you want.
``` ActionScript
var str3:String= stackpivot.toString();
flash.external.ExternalInterface.call("debug","stackpivot\n");
flash.external.ExternalInterface.call("debug",str3);
```

and then, you could use trick in hooking JavaScript function above

# How to add anchor in Flash, if decompilation is failed?
> Refer to [Check Point's Blog](https://blog.checkpoint.com/2017/05/05/debug-instrumentation-via-flash-actionscript/) and 
[MS's research](https://www.blackhat.com/docs/us-16/materials/us-16-Oh-The-Art-of-Reverse-Engineering-Flash-Exploits-wp.pdf)

Key idea is to leverage [RABCDAsm](https://github.com/CyberShadow/RABCDAsm) to disassemble flash, modify code, and then assemble to flash.


