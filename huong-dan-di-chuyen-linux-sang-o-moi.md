# Hướng dẫn chuyển Linux (root + home) sang ổ cứng mới

> Cập nhật: 2026-09-10 — số liệu trong tài liệu này lấy trực tiếp từ máy thật bằng `lsblk`, `df -h`, `cat /etc/fstab`, `swapon --show`.

---

## 1. Thông tin máy hiện tại (đã kiểm tra thực tế)

Máy: dual boot Windows + Ubuntu **20.04.6 LTS** (kernel 5.15.0-139-generic), UEFI, GPT, mainboard Gigabyte B760M-DS3H-DDR4.

### Ổ hệ thống hiện tại — `/dev/nvme0n1` (465,8 GiB ≈ 500GB)

| Partition | Size | FS | Mount | UUID | Ghi chú |
|---|---|---|---|---|---|
| `nvme0n1p1` | 128M | — | — | — | msftres (Microsoft reserved) |
| `nvme0n1p2` | 100M | vfat | `/boot/efi` | `08E9-08CD` | **ESP dùng chung Windows + Ubuntu** |
| `nvme0n1p3` | 339,3G | ntfs | — | `9CA2EA59A2EA3802` | Windows C: |
| `nvme0n1p4` | 711M | ntfs | — | `5C22EE7E22EE5C90` | Recovery / diag (hidden) |
| `nvme0n1p5` | 28,6G | ext4 | `/` | `ef810a03-8e85-4021-8b51-11ace42f0b26` | **root — dùng 26G, còn 1,1G, 97% đầy** |
| `nvme0n1p6` | 96,9G | ext4 | `/home` | `86287143-2a40-46f4-8337-ed0cd64b79ff` | **home — dùng 84G, còn 6,7G, 93% đầy** |

Swap: **không có swap partition**, dùng file `/swapfile` (1,3 GiB) nằm trong root.

### Ổ đích — `/dev/nvme1n1` (465,8 GiB ≈ 500GB)

| Partition | Size | Trạng thái |
|---|---|---|
| `nvme1n1p1` | 16M | msftres — **giữ nguyên, không xóa** |
| *(phần còn lại)* | **~465,7 GiB** | **unallocated — đã dọn xong, sẵn sàng dùng** |

✅ **Kết luận kiểm tra: chuyển được.** Tổng data cần copy = 26G (root) + 84G (home) = **110 GiB**, trong khi ổ mới trống **465,7 GiB** — dư gấp hơn 4 lần.

### Nội dung `/etc/fstab` hiện tại (bản gốc, để đối chiếu khi sửa)

```
UUID=ef810a03-8e85-4021-8b51-11ace42f0b26 /               ext4    errors=remount-ro 0       1
UUID=08E9-08CD                            /boot/efi       vfat    umask=0077        0       1
UUID=86287143-2a40-46f4-8337-ed0cd64b79ff /home           ext4    defaults          0       2
/swapfile                                 none            swap    sw                0       0
```

Sau khi chuyển ổ chỉ sửa **2 dòng đầu tiên của root và home**. Dòng `/boot/efi` và `/swapfile` **giữ nguyên không đổi**.

---

## 2. Chú ý quan trọng về đơn vị GB vs GiB (đọc trước khi chia partition)

Đây là chỗ dễ nhầm nhất, và là lý do kế hoạch "root 100GB + home 400GB" cần điều chỉnh:

- Nhà sản xuất ghi **500GB** = 500.000.000.000 byte (hệ thập phân).
- Linux/GParted hiển thị theo **GiB** (hệ nhị phân, 1 GiB = 1024³ byte).
- 500GB thập phân = **465,8 GiB** → đây là con số thật mà GParted sẽ cho bạn chia.

Nên **tổng root + home tối đa chỉ ~465,7 GiB**, không phải 500. Hai lựa chọn:

| Phương án | Root | Home | Ghi chú |
|---|---|---|---|
| **A (khuyến nghị)** | 100 GiB (`102400` MiB) | phần còn lại ≈ 365 GiB | Root rộng rãi, home vẫn gấp 3,8 lần hiện tại |
| B | 93 GiB (`95232` MiB) | ≈ 372 GiB (đúng 400GB thập phân) | Chỉ chọn nếu bạn muốn con số "400GB" đúng theo cách Windows hiển thị |

Tài liệu dưới đây dùng **phương án A**.

So sánh với hiện tại: root 28,6G → 100 GiB (gấp 3,5 lần), home 96,9G → ~365 GiB (gấp 3,8 lần). Giải quyết triệt để tình trạng root 97% và home 93% đầy.

---

⚠️ **Backup dữ liệu quan trọng ra ổ ngoài hoặc cloud trước khi bắt đầu.** Quy trình này là copy (không xóa nguồn) nên rủi ro thấp, nhưng thao tác partition luôn cần có đường lùi.

---

## 3. Bước 1 — Dọn ổ mới ✅ ĐÃ HOÀN THÀNH

Kiểm tra ngày 2026-09-10 cho thấy 2 partition NTFS (285GB và 3287MB) **đã được xóa xong**, ổ `nvme1n1` hiện chỉ còn `p1` (16M msftres) và ~465,7 GiB unallocated.

Không cần làm gì thêm ở bước này. (Nếu sau này cần lặp lại trên máy khác: mở Disk Management trong Windows → chuột phải từng volume NTFS → Delete Volume → giữ nguyên partition msftres.)

## 4. Bước 2 — Boot Live USB Ubuntu 20.04

Không thao tác resize/copy khi hệ thống đang chạy trực tiếp trên ổ nguồn — phải boot từ USB rời.

1. **Tạo USB boot:** dùng [Rufus](https://rufus.ie) (chạy trong Windows) hoặc Startup Disk Creator (trong Ubuntu), ghi file ISO **Ubuntu 20.04.x Desktop** (đúng version đang cài — 20.04.6 LTS) vào USB ≥ 8GB.
   - Tải ISO: https://releases.ubuntu.com/20.04/
   - Dùng đúng version rất quan trọng: khi chroot ở Bước 6, `grub-install` và `update-grub` chạy trên hệ thống 20.04 nên môi trường live cùng version tránh lệch phiên bản GRUB.
2. Cắm USB, khởi động lại, bấm phím vào Boot Menu ngay lúc màn hình logo hiện ra — mainboard Gigabyte B760M thường là **F12** (thử thêm F11 / ESC / Del nếu không vào được).
3. Chọn dòng có chữ **UEFI: `<tên USB>`** — không chọn dòng không có "UEFI" (tránh boot Legacy sai mode với ổ GPT).
4. Màn hình Ubuntu hiện lên, chọn **"Try Ubuntu"** (KHÔNG chọn "Install Ubuntu").
   - "Try Ubuntu" chạy hệ điều hành live từ RAM/USB, không cài gì lên máy, không mount ổ `nvme0n1` — đúng điều kiện an toàn để copy và sửa partition.
5. Vào được desktop live → mở **Terminal** (Ctrl+Alt+T) để làm các bước tiếp theo.

## 5. Bước 3 — Tạo partition mới bằng GParted

**GParted** (GNOME Partition Editor) là công cụ đồ họa quản lý phân vùng — tạo, xóa, resize, format. **Có sẵn trong Ubuntu Live USB, không cần cài thêm.**

Mở bằng một trong hai cách:
- Nhấn phím Super (Windows) → gõ `GParted` → Enter
- Terminal: `sudo gparted`

*(Trường hợp hiếm bản live thiếu: `sudo apt update && sudo apt install gparted`)*

**Thao tác:**

1. Góc **trên bên phải** có dropdown chọn ổ đĩa → chọn **`/dev/nvme1n1`**.
   > ⚠️ Kiểm tra kỹ: `nvme1n1` là ổ đích (chỉ có 1 partition 16M + vùng xám lớn). `nvme0n1` là ổ đang chạy hệ thống — **chọn nhầm ổ này là mất toàn bộ Windows + Linux hiện tại**. Nhận biết: `nvme1n1` không có partition nào màu ext4/ntfs lớn.
2. Thấy vùng lớn màu xám ghi "unallocated" (~465,7 GiB) nằm sau partition msftres 16M.
3. **Tạo partition root:** chuột phải vùng unallocated → **New**:
   - New size (MiB): **`102400`** (= 100 GiB)
   - File system: **`ext4`**
   - Label (tùy chọn): `root-new`
   - Create as: `Primary Partition`
   - Bấm **Add**
4. **Tạo partition home:** chuột phải phần unallocated còn lại → **New**:
   - New size: **để nguyên giá trị mặc định** (GParted tự điền toàn bộ dung lượng còn lại ≈ 374.000 MiB) — cách này tránh phải tính tay và tránh lỗi thiếu/thừa dung lượng
   - File system: **`ext4`**
   - Label (tùy chọn): `home-new`
   - Bấm **Add**
5. Bấm nút **✓ (Apply All Operations)** trên thanh công cụ → xác nhận → chờ vài phút cho tới khi báo "All operations successfully completed".
6. Đóng GParted.

Kết quả dự kiến: `nvme1n1p2` (100 GiB, ext4) và `nvme1n1p3` (~365 GiB, ext4). **Nhưng phải xác nhận lại bằng lệnh ở Bước 4** — không giả định số thứ tự.

## 6. Bước 4 — Xác nhận tên partition thật và mount nguồn/đích

> 🔴 **Đây là bước KHÔNG copy-paste mù được.** Phải đọc kết quả lệnh và thay tên partition thật vào.

Mở Terminal, chạy:

```bash
lsblk -f
```

Đọc kết quả, xác định chính xác tên 2 partition ext4 vừa tạo trên `nvme1n1` (thường là `nvme1n1p2` = root mới, `nvme1n1p3` = home mới — phân biệt bằng dung lượng: cái ~100G là root, cái ~365G là home).

Sau đó mount cả nguồn lẫn đích. Thay `nvme1n1pX` / `nvme1n1pY` bằng tên thật vừa đọc được:

```bash
sudo mkdir -p /mnt/oldroot /mnt/oldhome /mnt/newroot /mnt/newhome
sudo mount /dev/nvme0n1p5 /mnt/oldroot     # root cũ — tên này cố định, không đổi
sudo mount /dev/nvme0n1p6 /mnt/oldhome     # home cũ — tên này cố định, không đổi
sudo mount /dev/nvme1n1pX /mnt/newroot     # ⚠️ thay X = số partition root mới
sudo mount /dev/nvme1n1pY /mnt/newhome     # ⚠️ thay Y = số partition home mới
```

Kiểm tra đã mount đúng chưa trước khi copy:

```bash
lsblk -f | grep mnt
df -h /mnt/oldroot /mnt/oldhome /mnt/newroot /mnt/newhome
```

Phải thấy: `/mnt/oldroot` ~28G (dùng 26G), `/mnt/oldhome` ~95G (dùng 84G), `/mnt/newroot` ~100G (trống), `/mnt/newhome` ~365G (trống).

## 7. Bước 5 — Copy dữ liệu bằng rsync

Đoạn này **copy-paste nguyên, không cần sửa** (với điều kiện Bước 4 đã mount đúng).

**Copy root** (bỏ qua thư mục ảo của kernel, `/home` vì copy riêng, và `/swapfile` vì sẽ tạo lại):

```bash
sudo rsync -aAXH --info=progress2 /mnt/oldroot/ /mnt/newroot/ \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/*","/swapfile"}
```

**Copy home:**

```bash
sudo rsync -aAXH --info=progress2 /mnt/oldhome/ /mnt/newhome/
```

Giải thích cờ: `-a` archive (giữ permission, owner, timestamp, symlink), `-A` ACL, `-X` extended attributes, `-H` hard link, `--info=progress2` hiện % tiến độ tổng.

**Thời gian dự kiến:** với 110 GiB trên NVMe, khoảng 5–20 phút tùy tốc độ ổ. Chờ chạy xong hẳn, không tắt giữa chừng, không rút điện.

**Kiểm tra sau khi copy** (dung lượng hai bên phải xấp xỉ nhau):

```bash
sudo du -sh /mnt/oldhome /mnt/newhome
sudo du -sh --exclude=/swapfile /mnt/oldroot /mnt/newroot
```

*(Muốn chạy thử trước không ghi gì: thêm cờ `-n` vào lệnh rsync để dry-run.)*

## 8. Bước 6 — Chroot, sửa fstab, tạo lại swapfile, cài lại GRUB

### 8.1 Mount hệ thống ảo và chroot

Thay `nvme1n1pY` bằng partition home mới (giống Bước 4):

```bash
sudo mount --bind /dev  /mnt/newroot/dev
sudo mount --bind /dev/pts /mnt/newroot/dev/pts
sudo mount --bind /proc /mnt/newroot/proc
sudo mount --bind /sys  /mnt/newroot/sys
sudo mkdir -p /mnt/newroot/home
sudo mount /dev/nvme1n1pY /mnt/newroot/home        # ⚠️ thay Y
sudo mount /dev/nvme0n1p2 /mnt/newroot/boot/efi    # ESP vẫn ở ổ cũ, dùng chung

sudo chroot /mnt/newroot
```

Dấu nhắc terminal đổi thành `root@ubuntu:/#` nghĩa là đã vào chroot thành công.

### 8.2 Lấy UUID mới

```bash
blkid | grep nvme1n1
```

Kết quả sẽ có dạng (UUID của bạn sẽ khác):

```
/dev/nvme1n1p2: LABEL="root-new" UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="ext4" ...
/dev/nvme1n1p3: LABEL="home-new" UUID="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy" TYPE="ext4" ...
```

**Ghi lại 2 UUID này** (chép ra giấy hoặc mở thêm cửa sổ terminal) — sắp dùng ngay.

### 8.3 Sửa `/etc/fstab`

> 🔴 **Bước này phải gõ tay, không paste sẵn được.** Gõ sai UUID → hệ thống không boot được.

```bash
nano /etc/fstab
```

Sửa **đúng 2 dòng**:

| Dòng | UUID cũ (tìm chuỗi này) | Thay bằng |
|---|---|---|
| mount `/` | `ef810a03-8e85-4021-8b51-11ace42f0b26` | UUID của `nvme1n1pX` (root mới) |
| mount `/home` | `86287143-2a40-46f4-8337-ed0cd64b79ff` | UUID của `nvme1n1pY` (home mới) |

**Giữ nguyên không đổi:**
- Dòng `UUID=08E9-08CD /boot/efi vfat umask=0077 0 1` (ESP vẫn ở ổ cũ)
- Dòng `/swapfile none swap sw 0 0`

Lưu file: `Ctrl+O` → Enter → `Ctrl+X`.

Đối chiếu lại kết quả:

```bash
cat /etc/fstab
blkid | grep nvme1n1
```

So từng ký tự UUID hai bên trước khi đi tiếp.

### 8.4 Tạo lại swapfile

Vì đã exclude `/swapfile` lúc rsync, cần tạo lại (fstab vẫn còn dòng khai báo nó):

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
```

*(Đặt 2G thay vì 1,3G như cũ — ổ mới rộng rãi. `swapon` sẽ tự chạy khi boot theo fstab.)*

### 8.5 Cập nhật GRUB

```bash
update-grub
grub-install /dev/nvme1n1
exit
```

- `update-grub` sinh lại `grub.cfg` với UUID root mới và dò lại Windows qua os-prober.
- `grub-install /dev/nvme1n1` cài GRUB cho ổ mới (ghi vào ESP đang mount tại `/boot/efi`).
- `exit` thoát khỏi chroot.

## 9. Bước 7 — Unmount và reboot

```bash
sudo umount -R /mnt/newroot
sudo umount /mnt/oldroot /mnt/oldhome
sudo reboot
```

Rút USB khi máy khởi động lại. Ở GRUB menu chọn **Ubuntu**.

**Kiểm tra sau khi vào được desktop:**

```bash
df -h /
df -h /home
lsblk -f | grep -E "nvme1n1|nvme0n1p[56]"
swapon --show
```

Kết quả mong đợi:
- `/` → `/dev/nvme1n1pX`, size ~98G, dùng ~26G (còn ~67G trống)
- `/home` → `/dev/nvme1n1pY`, size ~359G, dùng ~84G (còn ~257G trống)
- `swapon --show` hiện `/swapfile 2G`
- `nvme0n1p5` và `nvme0n1p6` **không còn mount** (đã thành ổ dự phòng)

Kiểm tra thêm: khởi động lại lần nữa, vào GRUB chọn **Windows Boot Manager** để chắc chắn Windows vẫn boot bình thường.

## 10. Bước 8 — Dọn ổ cũ (làm sau vài ngày)

Giữ `nvme0n1p5` và `nvme0n1p6` nguyên vẹn **ít nhất vài ngày đến 1 tuần** làm backup. Khi đã chắc chắn hệ thống mới chạy ổn định:

- Mở GParted → chọn `/dev/nvme0n1` → xóa `p5` và `p6` → tạo partition mới (ext4 hoặc ntfs tùy nhu cầu) trên ~125G vừa giải phóng.
- Hoặc mở rộng phân vùng Windows `nvme0n1p3` sang vùng trống đó bằng Disk Management (chỉ mở rộng được nếu vùng trống nằm ngay sau partition C:).
- Sau khi xóa, chạy lại `sudo update-grub` để bỏ entry Ubuntu cũ khỏi menu GRUB.

---

## 11. Lưu ý và cảnh báo quan trọng

### Về khả năng boot

- ⚠️ **ESP vẫn nằm ở `nvme0n1p2` (ổ cũ).** Nghĩa là sau khi chuyển xong, **máy vẫn cần ổ cũ cắm sẵn để boot được** — file khởi động EFI của cả Ubuntu lẫn Windows đều ở đó. Rút ổ cũ ra là máy không boot.
- Muốn ổ mới boot hoàn toàn độc lập: cần tạo thêm 1 partition ESP (fat32, 512MB, cờ `boot`+`esp`) trên `nvme1n1`, mount nó vào `/boot/efi` khi chroot, sửa dòng `/boot/efi` trong fstab theo UUID ESP mới, rồi `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu /dev/nvme1n1`. Đây là hướng phức tạp hơn — chỉ làm nếu thật sự cần rút ổ cũ ra.

### Về rsync

- Quên `--exclude="/home/*"` khi copy root → toàn bộ 84G dữ liệu home bị copy lẫn vào root mới, ăn hết chỗ và gây trùng lặp. Đọc kỹ câu lệnh trước khi Enter.
- Quên `--exclude="/swapfile"` không gây lỗi nghiêm trọng, chỉ tốn thêm 1,3G và mất thời gian copy vô ích.
- Dấu `/` cuối đường dẫn nguồn (`/mnt/oldroot/`) là **bắt buộc** — thiếu nó rsync sẽ tạo thư mục lồng `/mnt/newroot/oldroot/` thay vì copy nội dung.

### Về fstab và UUID

- Gõ sai một ký tự UUID → boot vào màn hình `(initramfs)` hoặc emergency mode. **Cách cứu:** boot lại Live USB, chroot lại theo mục 8.1, sửa lại `/etc/fstab`, không cần làm lại từ đầu.
- Đừng dùng tên thiết bị (`/dev/nvme1n1p2`) thay cho `UUID=` trong fstab — tên thiết bị có thể đổi khi thêm/bớt ổ, UUID thì không.

### Về GParted

- **Luôn kiểm tra dropdown chọn ổ ở góc trên phải trước mỗi thao tác.** Thao tác nhầm trên `nvme0n1` (ổ đang chạy) là mất cả Windows lẫn Linux.
- Không thao tác trên partition đang được mount — nếu GParted báo partition có ổ khóa 🔒, phải unmount trước.

### Trường hợp cần làm lại

Tất cả các bước từ 4 đến 6 đều **không phá hủy dữ liệu ổ cũ** — `nvme0n1p5` và `nvme0n1p6` giữ nguyên suốt quá trình. Nếu bất kỳ bước nào lỗi, chỉ cần boot lại Live USB và làm lại; hệ thống cũ trên ổ cũ vẫn boot được bình thường cho tới khi bạn tự tay xóa ở Bước 8.

---

## 12. Bảng tóm tắt: bước nào copy-paste được, bước nào phải sửa tay

| Bước | Copy-paste được? | Cần sửa gì |
|---|---|---|
| 1 — Dọn ổ mới | ✅ Đã xong | — |
| 2 — Boot Live USB | Thao tác tay | Chọn "Try Ubuntu" |
| 3 — GParted | Thao tác tay (GUI) | Chọn đúng ổ `nvme1n1`, nhập `102400` MiB cho root |
| 4 — Mount | ⚠️ **Một phần** | Thay `nvme1n1pX` / `nvme1n1pY` bằng tên thật từ `lsblk -f` |
| 5 — rsync | ✅ Nguyên si | — |
| 6.1 — Chroot | ⚠️ **Một phần** | Thay `nvme1n1pY` (dòng mount home) |
| 6.2 — blkid | ✅ Nguyên si | Đọc và ghi lại 2 UUID |
| 6.3 — fstab | 🔴 **Gõ tay** | Thay 2 UUID root + home |
| 6.4 — swapfile | ✅ Nguyên si | — |
| 6.5 — GRUB | ✅ Nguyên si | — |
| 7 — Reboot | ✅ Nguyên si | — |
| 8 — Dọn ổ cũ | Thao tác tay | Làm sau vài ngày |

---

## 13. Link tham khảo

- [ArchWiki – GRUB (cài lại bootloader qua chroot, phần UEFI)](https://wiki.archlinux.org/title/GRUB)
- [Moving Arch Linux to a new SSD with rsync – rdeeson.com](https://www.rdeeson.com/weblog/157/moving-arch-linux-to-a-new-ssd-with-rsync)
- [Migrating a Linux system from one disk to another using chroot – itstorage.net](https://itstorage.net/index.php/ldce/laa/571-migrating-a-linux-system-from-one-disk-to-another-using-chroot)
- [Backup and restore with rsync – BunsenLabs forum](https://forums.bunsenlabs.org/viewtopic.php?id=3586)
- [Increase an ext4 partition with GParted – jesusamieiro.com](https://www.jesusamieiro.com/increase-an-ext4-partition-with-gparted/)
- [How To Resize Active/Primary root Partition Using GParted – 2DayGeek](https://www.2daygeek.com/linux-resize-active-primary-root-partition-gparted/)
- [Ubuntu 20.04 LTS releases (tải ISO cho Live USB)](https://releases.ubuntu.com/20.04/)
- [Rufus – tạo USB boot từ Windows](https://rufus.ie)
