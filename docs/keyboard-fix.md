# 键盘失灵解决方案

Xiaomi Book Pro 14 2026 在 Linux 下存在键盘失灵问题，具体表现为:

- 开机 / 休眠恢复后发生
- 仅 Fn 键有作用, Capslock 无作用，且灯不亮
- 其余按键均无作用

具体原因是: 键盘在 Linux 下被 `i8042` 驱动接管，然而 Linux 驱动仅通过 command byte 的 KBDDIS 位控制键盘启用，不会发送 0xAE（Enable Keyboard Interface）命令，然而小米 EC 固不响应 command byte 方式，需要显式 0xAE 命令。

因此解决方法是手动发送 0xAE 到端口 0x64 + 0xF4 到端口 0x60 激活内置键盘。

两个端口作用如下:

| 端口 | 用途 |
|:--:|:--:|
| `0x64` | PS/2 控制器命令端口 |
| `0x60` | PS/2 控制器数据端口 |

需要向两个端口发送命令，重新启用键盘：

1. **启用键盘接口**：向命令端口 `0x64` 写入 `0xAE`（Enable Keyboard Interface），使 PS/2 控制器重新开启键盘通道
2. **启用按键扫描**：向数据端口 `0x60` 写入 `0xF4`（Enable Scanning），通知键盘开始扫描按键输入

## 修复方法

可以使用你喜欢的编程方式实现上面的命令发送，并设置在开机 / 休眠唤醒时自动执行即可，比如下面的 Rust 代码:

```rust
use std::fs::OpenOptions;
use std::io::{Read, Seek, SeekFrom, Write};
use std::thread::sleep;
use std::time::Duration;

fn main() {
    let mut f = OpenOptions::new()
        .read(true)
        .write(true)
        .open("/dev/port")
        .expect("Failed to open /dev/port (need root)");

    // Send 0xAE (Enable Keyboard Interface) to command port 0x64
    f.seek(SeekFrom::Start(0x64)).unwrap();
    f.write_all(&[0xAE]).unwrap();
    sleep(Duration::from_millis(100));

    // Send 0xF4 (Enable Scanning) to data port 0x60
    f.seek(SeekFrom::Start(0x60)).unwrap();
    f.write_all(&[0xF4]).unwrap();
    sleep(Duration::from_millis(100));

    // Read status
    f.seek(SeekFrom::Start(0x64)).unwrap();
    let mut buf = [0u8; 1];
    f.read_exact(&mut buf).unwrap();
    println!("status: {:#04x}", buf[0]);
}
```


运行成功后会输出 PS/2 控制器状态寄存器的值，键盘应立即恢复工作。

### 设置 Systemd 服务自动启用键盘

将编译好的二进制文件复制到系统路径：

```bash
sudo cp target/release/fix-keyboard /usr/local/bin/
```

#### 开机启动

创建 systemd 服务文件 `/etc/systemd/system/fix-keyboard.service`：

```ini
[Unit]
Description=Fix Xiaomi Book Pro keyboard (re-enable PS/2 interface)
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/fix-keyboard

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable fix-keyboard.service
```

#### 休眠恢复后启动

创建 `/etc/systemd/system/fix-keyboard-resume.service`：

```ini
[Unit]
Description=Fix Xiaomi Book Pro keyboard after resume
After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/fix-keyboard

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable fix-keyboard-resume.service
```

## 替代方案

如果不想用 Rust 工具，也可以直接使用 Python 脚本实现相同功能：

```python
#!/usr/bin/env python3
import os, time

fd = os.open("/dev/port", os.O_RDWR)

# 0xAE: Enable Keyboard Interface
os.lseek(fd, 0x64, os.SEEK_SET)
os.write(fd, bytes([0xAE]))
time.sleep(0.1)

# 0xF4: Enable Scanning
os.lseek(fd, 0x60, os.SEEK_SET)
os.write(fd, bytes([0xF4]))
time.sleep(0.1)

os.close(fd)
print("Keyboard re-enabled.")
```

使用方式：

```bash
sudo python3 fix-keyboard.py
```

## 注意事项

- 此方案为临时缓解措施，只有通过小米更新 EC 固件或者 Linux 上游更新驱动代码才能彻底解决
- 工具需要 root 权限才能访问 `/dev/port`
- 如果内核未启用 `/dev/port`，需要确保内核配置中 `CONFIG_DEVPORT=y`
