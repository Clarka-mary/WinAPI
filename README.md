winreg [![Crates.io](https://img.shields.io/crates/v/winreg.svg)](https://crates.io/crates/winreg)
======

Rust bindings to MS Windows Registry API. Work in progress.

Current features:
* Basic registry operations:
    * open/create/delete keys
    * read and write values
    * seamless conversion between `REG_*` types and rust primitives
        * `String` <= `REG_SZ`, `REG_EXPAND_SZ` or `REG_MULTI_SZ`
        * `String` and `&str` => `REG_SZ`
        * `u32` <=> `REG_DWORD`
        * `u64` <=> `REG_QWORD`
* Iteration through key names and through values
* Serialization of rust types into/from registry (only primitives and structures for now)

## Usage

### Basic usage

```rust
extern crate winreg;
use std::path::Path;
use winreg::RegKey;
use winreg::enums::*;

fn main() {
    println!("Reading some system info...");
    let hklm = RegKey::predef(HKEY_LOCAL_MACHINE);
    let cur_ver = hklm.open_subkey_with_flags("SOFTWARE\\Microsoft\\Windows\\CurrentVersion",
        KEY_READ).unwrap();
    let pf: String = cur_ver.get_value("ProgramFilesDir").unwrap();
    let dp: String = cur_ver.get_value("DevicePath").unwrap();
    println!("ProgramFiles = {}\nDevicePath = {}", pf, dp);
    let info = cur_ver.query_info().unwrap();
    println!("info = {:?}", info);

    println!("And now lets write something...");
    let hkcu = RegKey::predef(HKEY_CURRENT_USER);
    let path = Path::new("Software").join("WinregRsExample1");
    let key = hkcu.create_subkey(&path).unwrap();

    key.set_value("TestSZ", &"written by Rust").unwrap();
    let sz_val: String = key.get_value("TestSZ").unwrap();
    key.delete_value("TestSZ").unwrap();
    println!("TestSZ = {}", sz_val);

    key.set_value("TestDWORD", &1234567890u32).unwrap();
    let dword_val: u32 = key.get_value("TestDWORD").unwrap();
    println!("TestDWORD = {}", dword_val);

    key.set_value("TestQWORD", &1234567891011121314u64).unwrap();
    let qword_val: u64 = key.get_value("TestQWORD").unwrap();
    println!("TestQWORD = {}", qword_val);

    key.create_subkey("sub\\key").unwrap();
    hkcu.delete_subkey_all(&path).unwrap();

    println!("Trying to open nonexisting key...");
    println!("{:?}", hkcu.open_subkey(&path).unwrap_err());
}
```

### Iterators

```rust
extern crate winreg;
use winreg::RegKey;
use winreg::enums::*;

fn main() {
    println!("File extensions, registered in system:");
    for i in RegKey::predef(HKEY_CLASSES_ROOT)
        .enum_keys().map(|x| x.unwrap())
        .filter(|x| x.starts_with("."))
    {
        println!("{}", i);
    }

    let system = RegKey::predef(HKEY_LOCAL_MACHINE)
        .open_subkey_with_flags("HARDWARE\\DESCRIPTION\\System", KEY_READ)
        .unwrap();
    for (name, value) in system.enum_values().map(|x| x.unwrap()) {
        println!("{} = {:?}", name, value);
    }
}
```

### Serialization

```rust
extern crate rustc_serialize;
extern crate winreg;
use winreg::enums::*;

#[derive(Debug,RustcEncodable,RustcDecodable,PartialEq)]
struct Rectangle{
    x: u32,
    y: u32,
    w: u32,
    h: u32,
}

#[derive(Debug,RustcEncodable,RustcDecodable,PartialEq)]
struct Test {
    t_bool: bool,
    t_u8: u8,
    t_u16: u16,
    t_u32: u32,
    t_u64: u64,
    t_usize: usize,
    t_struct: Rectangle,
    t_string: String,
    t_i8: i8,
    t_i16: i16,
    t_i32: i32,
    t_i64: i64,
    t_isize: isize,
    t_f64: f64,
    t_f32: f32,
}

fn main() {
    let hkcu = winreg::RegKey::predef(HKEY_CURRENT_USER);
    let key = hkcu.create_subkey("Software\\RustEncode").unwrap();
    let v1 = Test{
        t_bool: false,
        t_u8: 127,
        t_u16: 32768,
        t_u32: 123456789,
        t_u64: 123456789101112,
        t_usize: 123456789101112,
        t_struct: Rectangle{
            x: 55,
            y: 77,
            w: 500,
            h: 300,
        },
        t_string: "test 123!".to_string(),
        t_i8: -123,
        t_i16: -2049,
        t_i32: 20100,
        t_i64: -12345678910,
        t_isize: -1234567890,
        t_f64: -0.01,
        t_f32: 3.14,
    };

    key.encode(&v1).unwrap();

    let v2: Test = key.decode().unwrap();
    println!("Decoded {:?}", v2);

    // This shows `false` because f32 and f64 encoding/decoding is NOT precise
    println!("Equal to encoded: {:?}", v1 == v2);
}
```
