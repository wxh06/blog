---
title: 使用 macOS 在 Arduino UNO 上运行 Rust
date: 2023-10-10 19:25:45 +0800
category: embedded
tags: arduino rust macos
locale: zh_CN
---

笔者在学校参加了一个 Arduino 社团，社团里面教的是使用官方的 [Arduino IDE](https://www.arduino.cc/en/software)。但作为一个合格的 Rust 批，我怎么能用 C 来进行嵌入式编程呢？

### 安装依赖

这里假设读者使用 rustup 安装过工具链。有一段时间没碰过 Rust 了，先更新一下：

```sh
rustup update
```

我们需要使用 Homebrew 与 Cargo 安装依赖：

```sh
brew install osx-cross/avr/avr-gcc avrdude
cargo install ravedude cargo-generate
```

- `avr-gcc` - 编译工具链
- `avrdude` - AVR Downloader Uploader，用于将程序上传到板子
- [`ravedude`](https://crates.io/crates/ravedude) - 是 `avrdude` 的 wrapper
- [`cargo-generate`](https://crates.io/crates/cargo-generate) - 用于稍后的使用模版创建项目（optional）

### 新建项目

```sh
cargo generate --git https://github.com/Rahix/avr-hal-template.git
```

按照 cargo-generate 的提示创建好项目后，`cd` 进目录或使用你偏好的编辑器打开文件夹。

目录结构如下所示：

```text
.
├── Cargo.lock
├── Cargo.toml
├── LICENSE-APACHE
├── LICENSE-MIT
├── README.md
├── avr-specs
│   ├── avr-atmega1280.json
│   ├── avr-atmega168.json
│   ├── avr-atmega2560.json
│   ├── avr-atmega328p.json
│   ├── avr-atmega32u4.json
│   ├── avr-atmega48p.json
│   ├── avr-attiny85.json
│   └── avr-attiny88.json
├── rust-toolchain.toml
└── src
    └── main.rs
```

此时 `src/main.rs` 中包含一个以一秒为间隔闪烁 L LED 灯的程序。

### 编译

```sh
$ cargo build
    Finished dev [optimized + debuginfo] target(s) in 0.05s
```

> 在每次修改完代码之后不需要单独运行 `cargo build`，接下来介绍的 `cargo run` 会在将程序上传到 Arduino 前自动进行编译。

### 运行

使用 USB 连接 Arduino 后，我们首先需要找到串口的文件路径。

```sh
$ ls /dev/*.usbserial-*
/dev/cu.usbserial-10  /dev/tty.usbserial-10
```

> 关于在 `cu` 与 `tty` 之间如何选择，可以参考 [Choosing between /dev/tty.usbserial vs /dev/cu.usbserial - Stack Overflow](https://stackoverflow.com/questions/37688257/choosing-between-dev-tty-usbserial-vs-dev-cu-usbserial)。

在执行 `cargo run` 时，我们需要设置环境变量 `RAVEDUDE_PORT` 或使用 `-P` flag 来指定 Arduino 的串口：

```sh
cargo run -- -P /dev/cu.usbserial-10
# or
RAVEDUDE_PORT=/dev/cu.usbserial-10 cargo run
```

可以使用 `export` 来设置环境变量，这样在当前 TTY 中后续每次只需要运行单独的 `cargo run` 命令即可：

```sh
export RAVEDUDE_PORT=/dev/cu.usbserial-10
cargo run
```

可以看到 Arduino UNO 上的 L LED 按我们预期工作了！

### 参考资料

- [A complete guide to running Rust on Arduino](https://blog.logrocket.com/complete-guide-running-rust-arduino/)
- [如何在 Arduino Uno 中运行 Rust：让你的 LED 灯闪起来](https://n3xtchen.github.io/n3xtchen/rust/2020/08/22/rust-arduino-our-first-blink)
