![Willhook logo](https://github.com/myood/willhook-rs/raw/main/willhook.png)

![Build](https://github.com/myood/willhook-rs/actions/workflows/build.yml/badge.svg) ![Docs.rs](https://img.shields.io/docsrs/willhook/latest?label=Docs.rs&logo=Docs.rs) ![issues](https://img.shields.io/github/issues/myood/willhook-rs)


Capture all keyboard and mouse input regardless of focus on the active windows desktop

# What this crate provides

This Windows-only crate provides safe and correct means to listen for keyboard and mouse events regardless of application focus.
The application can be CLI or with a Window.

Under the hood the crate leverages the **WI**ndows **L**ow-**L**evel **HOOK**s.
You can read more about that topic on [MSDN](https://learn.microsoft.com/en-us/windows/win32/winmsg/about-hooks?redirectedfrom=MSDN).
The crate was created for learning-purposes mostly and for my hobby project, but we will see where it goes.

The design goals for this crate are to be: correct, misuse-proof and fail-proof.
Having that in mind, the implementation follows best effort to avoid any panic.
In the worst case, it should just return incomplete input event (e.g. with missing keyboard key code).

### What this crate does NOT provide

This crate is intended for "read-only" access to hooks. It does not support injecting input events or altering them.
If you are looking for that kind of functionality, you can give [mki](https://crates.io/crates/mki) a try.
In comparison, the mki crate supports also Linux, but does not cleanup the low-level hooks (by [unhooking them](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-unhookwindowshookex)) and threads behind them (by [joinging with them](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html#method.join)).
This *may* not be an issue for you. The addition of "injecting" and "altering" input events to [willhook] is a possibility, although it is not top priority.

# Warning: The current state

Currently it supports listening to mouse and keyboard actions, see [event] module for details.
There is no fancy logic to interpret the events - with this crate you can just received them and do what you want with that information.
In that aspect, I consider it feature complete.
There are integration tests that should cover all realistic scenarios.
There are also some unit tests covering less realistic cases, when Windows OS would send invalid input.
I think the crate is rather well tested, but still, keep in mind that the crate is also "young".
Note: the integration tests inject mouse and keyboard events, also they need to be run sequentially (no multi-threading). 
There are some tests that do not pass on GitHub Actions and are ignored.
With that in mind, run the tests with `cargo test --tests -- --test-threads=1 --include-ignored`.
*It is highly recommended to at least quickly review the code before using this crate for anything more then hobby projects, at least at the current state.*

TODO:
- document unsafe code before I forget all the quirks :-)
- maybe write more unit tests
- maybe improve the crate partitioning to modules (without breaking the API)
- maybe rework underlying channels, so that they are dropped with the hook (now they are just drained)
- maybe add injecting events
- maybe add blocking events, if even possible
- maybe add manipulating events, if even possible

# How it works

In short, there are a few handy functions to request a hook: [keyboard_hook], [mouse_hook] and [willhook].
When called they:
- start background thread(s) for each low-level hook, and in that thread(s):
    - register a mouse and/or keyboard low-level hook(s)
    - start Windows message queue and wait for the message to end execution
- create, if were not created already, the channels for passing events to "client" thread
- return the handle to the underlying low-level hooks as [hook::Hook]

When the [hook::Hook] goes out of scope, the underlying resources supporting low-level hooks are dropped:
- each of the underlying low-level hooks is unhooked from the Windows Kernel
- each of the background threads is properly joined
- all pending events are dropped (background channels are drained)

When the [hook::Hook] is active (in scope / not dropped). 
Then one can receive recorded [event::InputEvent]s via [hook::Hook::try_recv].
It works similiarly to [std::sync::mpsc::Receiver::try_recv].

# Quick example

```rust
use willhook::willhook;
use std::sync::{Arc, atomic::{Ordering, AtomicBool}};

fn main() {
    let is_running = Arc::new(AtomicBool::new(true));
    let set_running = is_running.clone();

    let h = willhook().unwrap();

    ctrlc::set_handler(move || {
        set_running.store(false, Ordering::SeqCst);
    })
    .expect("Error setting Ctrl-C handler");

    while is_running.load(Ordering::SeqCst) {
        if let Ok(ie) = h.try_recv() {
            match ie {
                willhook::InputEvent::Keyboard(ke) => println!("{:?}", ke),
                willhook::InputEvent::Mouse(me) => println!("{:?}", me),
                _ => println!("Input event: {:?}", ie),
            }
        } else {
            std::thread::yield_now();   
        }
    };
}

```

Example output:

```
PS ~ cargo run --example showcase
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target\debug\examples\showcase.exe`
KeyboardEvent { pressed: Down(Normal), key: Some(A), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Up(Normal), key: Some(A), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Down(Normal), key: Some(Q), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Up(Normal), key: Some(Q), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Down(System), key: Some(LeftAlt), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Down(System), key: Some(A), is_injected: Some(NotInjected) }
KeyboardEvent { pressed: Up(System), key: Some(A), is_injected: Some(NotInjected) }
MouseEvent { event: Press(MousePressEvent { pressed: Down, button: Left(SingleClick) }), is_injected: Some(NotInjected) }
MouseEvent { event: Press(MousePressEvent { pressed: Up, button: Left(SingleClick) }), is_injected: Some(NotInjected) }
MouseEvent { event: Move(MouseMoveEvent { point: Some(Point { x: 1010, y: 1188 }) }), is_injected: Some(NotInjected) }
MouseEvent { event: Move(MouseMoveEvent { point: Some(Point { x: 1013, y: 1188 }) }), is_injected: Some(NotInjected) }
MouseEvent { event: Wheel(MouseWheelEvent { wheel: Vertical, direction: Some(Backward) }), is_injected: Some(NotInjected) }
MouseEvent { event: Wheel(MouseWheelEvent { wheel: Vertical, direction: Some(Forward) }), is_injected: Some(NotInjected) }
MouseEvent { event: Move(MouseMoveEvent { point: Some(Point { x: 1068, y: 1189 }) }), is_injected: Some(NotInjected) }
MouseEvent { event: Move(MouseMoveEvent { point: Some(Point { x: 1067, y: 1189 }) }), is_injected: Some(NotInjected) }
MouseEvent { event: Press(MousePressEvent { pressed: Down, button: Middle(SingleClick) }), is_injected: Some(NotInjected) }
```
