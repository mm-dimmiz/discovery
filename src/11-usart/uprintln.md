# `uprintln!`

For the next exercise, we'll implement the `uprint!` family of macros. Your goal is to make this
line of code work:

``` rust
    uprintln!("The answer is {}", 40 + 2)
```

Which must send the string `"The answer is 42"` through the serial interface.

How do we go about that? It's informative to look into the `std` implementation of `println!`.

``` rust
// src/libstd/macros.rs
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print(format_args!($($arg)*)));
}
```

Looks simple so far. We need the built-in `format_args!` macro (it's implement in the compiler so we
can't see what it actually does). We'll have to use that macro in the exact same way. What does this
`_print` function do?

``` rust
// src/libstd/io/stdio.rs
pub fn _print(args: fmt::Arguments) {
    let result = match LOCAL_STDOUT.state() {
        LocalKeyState::Uninitialized |
        LocalKeyState::Destroyed => stdout().write_fmt(args),
        LocalKeyState::Valid => {
            LOCAL_STDOUT.with(|s| {
                if s.borrow_state() == BorrowState::Unused {
                    if let Some(w) = s.borrow_mut().as_mut() {
                        return w.write_fmt(args);
                    }
                }
                stdout().write_fmt(args)
            })
        }
    };
    if let Err(e) = result {
        panic!("failed printing to stdout: {}", e);
    }
}
```

That *looks* complicated but the only part we are interested in is: `w.write_fmt(args)` and
`stdout().write_fmt(args)`. What `print!` ultimately does is call the `fmt::Write::write_fmt` method
with the output of `format_args!` as its argument.

Luckily we don't have to implement the `fmt::Write::write_fmt` method either because it's a default
method. We only have to implement the `fmt::Write::write_str` method.

Let's do that.

This is what the macro side of the equation looks like. What's left to be done by you is provide the
implementation of the `write_str` method.

Above we saw that `Write` is in `std::fmt`. We don't have access to `std` but `Write` is also
available in `core::fmt`.

``` rust
#![no_std]

extern crate aux11;

use core::fmt::{self, Write};

use aux11::usart1;

macro_rules! uprint {
    ($serial:expr, $($arg:tt)*) => {
        $serial.write_fmt(format_args!($($arg)*)).ok()
    };
}

macro_rules! uprintln {
    ($serial:expr, $fmt:expr) => {
        uprint!($serial, concat!($fmt, "\n"))
    };
    ($serial:expr, $fmt:expr, $($arg:tt)*) => {
        uprint!($serial, concat!($fmt, "\n"), $($arg)*)
    };
}

struct SerialPort {
    usart1: &'static mut usart1::RegisterBlock,
}

impl Write for SerialPort {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        // TODO implement this
        // hint: this will look very similar to the previous program
    }
}

fn main() {
    let (usart1, _mono_timer, _itm) = aux11::init();

    let mut serial = SerialPort { usart1 };

    uprintln!(serial, "The answer is {}", 40 + 2);
}
```
