# Chrome IPC Messages

Question:

`打开一个网页，chrome的browser process与render process之间的messages有哪些？各自做什么事情？`

## 

任何一种MessageFilter都是

```cpp
// A helper interface which owns an associated interface binding on the IO
// thread. Subclassess of BrowserMessageFilter may use this to simplify
// the transition to Mojo interfaces.
//
// In general the correct pattern for using this is as follows:
//
//   class FooMessageFilter : public BrowserMessageFilter,
//                            public BrowserAssociatedInterface<mojom::Foo>,
//                            public mojom::Foo {
//    public:
//     FooMessageFilter()
//         : BrowserMessageFilter(FooMsgStart),
//           BrowserAssociatedInterface<mojom::Foo>(this, this) {}
//
//     // BrowserMessageFilter implementation:
//     bool OnMessageReceived(const IPC::Message& message) override {
//       // ...
//       return true;
//     }
//
//     // mojom::Foo implementation:
//     void DoStuff() override { /* ... */ }
//   };
//
// The remote side of an IPC channel can request the |mojom::Foo| associated
// interface and use it would use any other associated interface proxy. Messages
// received for |mojom::Foo| on the local side of the channel will retain FIFO
// with respect to classical IPC messages received via OnMessageReceived().
//
// See BrowserAssociatedInterfaceTest.Basic for a simple working example usage.
template <typename Interface>
class BrowserAssociatedInterface {
}
```