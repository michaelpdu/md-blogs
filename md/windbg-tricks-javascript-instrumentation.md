* JavaScript Instrumentation

Refer to [d0c_s4vage's blog](http://d0cs4vage.blogspot.hk/2013/06/windbg-tricks-javascript-windbg.html)

A better way to instrument windbg via javascript is to create a way for javascript to print a message in windbg (and trigger a break): 
```
bu jscript!Js::Math::Atan ".printf \"DEBUG: %mu\\n\", poi(poi(esp+10)+c) ; g"
```
(If you want to break, remove the ; g)

That's cool, but what if you want to do something a little more complicated, like track all allocations of a specific size after certain javascript statements have been executed. With the previous method, the javascript would have to look something like this:
```
function log(msg) {
    Math.atan(msg);
}

function track_all_allocations_and_frees_size_x20() {
    Math.asin();
}

log("Executing main javascript");
execute_main_javascript();

log("Track all allocations and frees now");
track_all_allocations_and_frees_size_x20();
do_something_cool();
```
... and the windbg breakpoints would be something like this:
```
bu jscript!Js::Math::Atan ".printf \"DEBUG: %mu\\n\", poi(poi(esp+10)+c) ; g"
bu jscript!Js::Math::Asin "bp ntdll!RtlAllocateHeap .if(poi(esp+c) == 0x20) { .echo ALLOCATED ONE ; knL } ; g"
```
This is more useful, but is still very inflexible. For every new javascript<-->windbg binding you might want, you'd need to also modify your breakpoints in windbg.
