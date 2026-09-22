# Hướng dẫn chuyển Linux (root + home) sang ổ cứng mới

> Cập nhật: 2026-09-11 — đã hoàn thành di chuyển thật trên máy. Tài liệu này giờ vừa là **báo cáo kết quả** vừa là **quy trình tham khảo** nếu cần lặp lại trên máy khác.

---

## ✅ TÌNH TRẠNG HIỆN TẠI: ĐÃ HOÀN THÀNH, ĐÃ XÁC NHẬN CHẠY ỔN ĐỊNH

Kiểm tra thực tế trên máy (2026-09-11):

| Mount | Partition | Size | Used | Avail | Use% |
|---|---|---|---|---|---|
| `/` | `/dev/nvme1n1p2` (label `root-robotics`) | 98G | 26G | 68G | 28% |
| `/home` | `/dev/nvme1n1p3` (label `home-robotics`) | 357G | 84G | 255G | 25% |
| `/boot/efi` | `/dev/nvme1n1p4` (vfat, UUID `DB34-FDB5`) | ~1G | — | 1015,9M | 1% |

Swap: `/swapfile` 1,3G, `swapon --show` xác nhận đang active.

**Điểm quan trọng nhất: ESP (EFI System Partition) nằm ngay trên ổ mới (`nvme1n1p4`), KHÔNG còn phụ thuộc ổ cũ `nvme0n1`.** Boot entry UEFI đã tạo:

```
Boot0000* ubuntu → nvme1n1p4 → \EFI\ubuntu\shimx64.efi
```

→ Nghĩa là **ổ mới đã boot độc lập hoàn toàn** — đây là kết quả tốt hơn kế hoạch ban đầu (kế hoạch cũ dùng chung ESP với ổ cũ, xem mục "Lịch sử thay đổi kế hoạch" bên dưới). Ổ cũ (`nvme0n1p5`, `nvme0n1p6`) hiện không còn mount, giữ làm bản dự phòng, có thể dọn theo Bước 8.

So với trạng thái trước khi chuyển (root 97% đầy, home 93% đầy) → vấn đề thiếu dung lượng đã giải quyết triệt để.

---

## 1. Thông tin máy (trước khi chuyển, để đối chiếu lịch sử)

Máy: dual boot Windows + Ubuntu **20.04.6 LTS** (kernel 5.15.0-139-generic), UEFI, GPT, mainboard Gigabyte B760M-DS3H-DDR4.

### Ổ hệ thống cũ — `/dev/nvme0n1` (465,8 GiB ≈ 500GB) — vẫn còn, đã hết vai trò boot

| Partition | Size | FS | UUID | Ghi chú |
|---|---|---|---|---|
| `nvme0n1p1` | 128M | — | — | msftres |
| `nvme0n1p2` | 100M | vfat | `08E9-08CD` | ESP cũ, không còn dùng để boot nữa |
| `nvme0n1p3` | 339,3G | ntfs | `9CA2EA59A2EA3802` | Windows C: |
| `nvme0n1p4` | 711M | ntfs | `5C22EE7E22EE5C90` | Recovery/diag (hidden) |
| `nvme0n1p5` | 28,6G | ext4 | `ef810a03-8e85-4021-8b51-11ace42f0b26` | root cũ — **backup, không mount** |
| `nvme0n1p6` | 96,9G | ext4 | `86287143-2a40-46f4-8337-ed0cd64b79ff` | home cũ — **backup, không mount** |

### Ổ mới — `/dev/nvme1n1` (465,8 GiB) — layout thật đã dùng

| Partition | Size | FS | Label | Mount |
|---|---|---|---|---|
| `nvme1n1p1` | 16M | — | — | msftres, giữ nguyên |
| `nvme1n1p2` | 100G | ext4 | `root-robotics` | `/` |
| `nvme1n1p3` | 364G | ext4 | `home-robotics` | `/home` |
| `nvme1n1p4` | 1G | vfat | — (ESP) | `/boot/efi` |

---

## 2. Lịch sử thay đổi kế hoạch (đọc để hiểu vì sao khác bản nháp ban đầu)

Bản kế hoạch đầu tiên (soạn trước khi thực hiện) dự tính:
- Root 100 GiB + home ~365 GiB, **dùng chung ESP với ổ cũ** (đơn giản hơn nhưng ổ mới không boot độc lập được).

Khi thực hiện thật, quyết định tạo luôn **ESP riêng trên ổ mới** (`nvme1n1p4`, 1GB) — đây là phần "nâng cao/tùy chọn" trong bản nháp cũ, nay là cách đã dùng thật và đã xác nhận hoạt động. Kích thước cuối cùng: root 100G, home 364G (home nhỏ hơn dự tính ~365G một chút vì trừ đi 1G cho ESP mới).

Về swap: bản nháp cũ dự tính exclude `/swapfile` lúc rsync rồi tạo lại 2G. Thực tế **không cần làm vậy** — `/swapfile` được rsync copy thẳng như file thường, `swapon --show` xác nhận vẫn hoạt động bình thường ở kích thước gốc 1,3G. Đơn giản hơn, không cần bước tạo lại.

---

## 3. Quy trình đầy đủ (đã xác nhận đúng qua thực hiện thật)

### Bước 1 — Dọn ổ mới ✅

Xóa NTFS trên ổ mới qua Windows Disk Management, giữ nguyên `msftres`. (Đã làm xong từ đầu.)

### Bước 2 — Boot Live USB Ubuntu 20.04

USB Ubuntu 20.04.x Desktop (đúng version máy đang chạy), boot UEFI, chọn **"Try Ubuntu"**. Phím vào boot menu trên mainboard Gigabyte B760M: **F12**.

### Bước 3 — Tạo partition bằng GParted (bao gồm ESP riêng)

Trên `/dev/nvme1n1`, tạo theo đúng thứ tự — **ESP phải tạo trước hoặc sau đều được nhưng phải nằm trong 4 partition này**:

1. Root: ext4, `102400` MiB (~100 GiB)
2. Home: ext4, phần lớn còn lại (~364 GiB — chừa 1GB cuối cho ESP)
3. ESP: fat32, `1024` MiB, **bật cờ `esp` + `boot`** (GParted: chuột phải partition này sau khi tạo → Manage Flags → tick `esp`)

> ⚠️ Cờ `esp`/`boot` bắt buộc phải bật thì UEFI firmware mới nhận ra đây là partition boot hợp lệ.

Apply, đóng GParted. Xác nhận lại bằng `lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT /dev/nvme1n1` — không dùng `MOUNTPOINTS` (số nhiều), bản lsblk trong live image không hỗ trợ cột này, gây lỗi `unknown column: MOUNTPOINTS`. Dùng `MOUNTPOINT` (số ít).

### Bước 4 — Mount nguồn và đích

```bash
sudo mkdir -p /mnt/oldroot /mnt/oldhome /mnt/new_root
sudo mount /dev/nvme0n1p5 /mnt/oldroot
sudo mount /dev/nvme0n1p6 /mnt/oldhome
sudo mount /dev/nvme1n1p2 /mnt/new_root          # partition root mới — xác nhận số thật bằng lsblk trước
sudo mkdir -p /mnt/new_root/home /mnt/new_root/boot/efi
sudo mount /dev/nvme1n1p3 /mnt/new_root/home     # partition home mới
sudo mount /dev/nvme1n1p4 /mnt/new_root/boot/efi # ESP mới
```

*(Ghi chú: dùng tên `/mnt/new_root` — có gạch dưới — nhất quán với các lệnh unmount troubleshooting ở mục 4 bên dưới.)*

### Bước 5 — Copy dữ liệu bằng rsync

```bash
sudo rsync -aAXH --info=progress2 /mnt/oldroot/ /mnt/new_root/ \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/*","/boot/efi/*"}

sudo rsync -aAXH --info=progress2 /mnt/oldhome/ /mnt/new_root/home/
```

`/swapfile` **không cần exclude** — copy thẳng theo root như file thường, hoạt động lại bình thường sau khi boot, không cần tạo lại bằng `fallocate`/`mkswap`.

### Bước 6 — Chroot, sửa fstab, cài GRUB vào ESP mới

```bash
sudo mount --bind /dev      /mnt/new_root/dev
sudo mount --bind /dev/pts  /mnt/new_root/dev/pts
sudo mount --bind /proc     /mnt/new_root/proc
sudo mount --bind /sys      /mnt/new_root/sys
sudo chroot /mnt/new_root
```

Trong chroot:

```bash
blkid | grep nvme1n1          # ghi lại UUID root mới, home mới, ESP mới (fat32 dùng UUID dạng ngắn XXXX-XXXX)
nano /etc/fstab               # sửa 3 dòng: /, /home, /boot/efi theo UUID mới; dòng /swapfile giữ nguyên
update-grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu /dev/nvme1n1
exit
```

`--bootloader-id=ubuntu` chính là tên tạo ra entry `Boot0000* ubuntu` xác nhận được sau khi boot lại.

### Bước 7 — Unmount trước khi reboot (điểm hay vướng lỗi "target is busy")

**Không dùng `sudo umount /mnt/new_root` ngay** — sẽ báo lỗi `target is busy` vì còn mount con bên trong (`dev`, `proc`, `sys`, và `efivarfs` do chroot tự mount thêm khi chạy `grub-install`). Phải tháo **đúng thứ tự từ trong ra ngoài**:

```bash
sudo umount /mnt/new_root/sys/firmware/efi/efivars
sudo umount /mnt/new_root/sys
sudo umount /mnt/new_root/proc
sudo umount /mnt/new_root/dev/pts
sudo umount /mnt/new_root/dev
sudo umount /mnt/new_root/boot/efi
sudo umount /mnt/new_root/home
sudo umount /mnt/new_root
sudo umount /mnt/oldroot /mnt/oldhome
```

Kiểm tra sạch trước khi reboot:

```bash
mount | grep '/mnt/new_root'     # không được ra dòng nào
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT /dev/nvme1n1
```

Nếu `mount | grep` vẫn còn dòng nào → **dừng lại, đừng dùng `-f` hay `-l` hay reboot ngay** — xử lý tiếp dòng đó trước (thường do quên tháo `efivarfs`, đây là mount hay sót nhất vì nó nằm sâu trong `sys/firmware/efi/efivars`).

```bash
sudo reboot
```

Rút USB lúc máy khởi động lại.

### Bước 8 — Xác nhận sau khi boot vào hệ thống mới

```bash
findmnt /
findmnt /home
findmnt /boot/efi
lsblk -f
swapon --show
```

Kết quả đúng: `/` → `nvme1n1p2`, `/home` → `nvme1n1p3`, `/boot/efi` → `nvme1n1p4`. **Đây chính là trạng thái đã xác nhận đạt được** (xem bảng ở đầu tài liệu).

Kiểm tra thêm: reboot lần nữa, vào GRUB chọn **Windows Boot Manager** — xác nhận Windows vẫn boot bình thường qua ESP cũ trên `nvme0n1p2` (Windows không đổi ESP, không liên quan tới việc Ubuntu đã chuyển sang ESP riêng).

### Bước 9 — Dọn ổ cũ (làm sau, không vội)

`nvme0n1p5`/`p6` đang là bản dự phòng, giữ ít nhất vài ngày–1 tuần trước khi xóa/format lấy lại ~125G. `nvme0n1p2` (ESP cũ) không cần đụng — Windows vẫn dùng.

---

## 4. Lưu ý và cảnh báo (rút ra từ quá trình thực hiện thật)

### Lỗi cú pháp lệnh đã gặp và cách sửa

- `lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS` → lỗi `unknown column: MOUNTPOINTS`. **Đúng phải là `MOUNTPOINT`** (số ít) trên phiên bản `lsblk` trong Ubuntu 20.04 live image.

### Lỗi "target is busy" khi umount

- Nguyên nhân: mount `--bind` cho `dev/proc/sys` và `efivarfs` (tự sinh khi chroot đụng tới `/sys/firmware/efi/efivars`, ví dụ lúc `grub-install`) vẫn còn treo bên trong `/mnt/new_root` dù đã umount `/mnt/new_root` chính.
- Cách phát hiện: `mount | grep '/mnt/new_root'` — liệt kê hết các dòng còn treo.
- Cách sửa: umount **từng dòng theo thứ tự từ sâu nhất ra ngoài** (xem Bước 7), **không dùng `umount -f`/`-l` để né lỗi** — làm vậy có thể để sót tiến trình đang giữ file, rủi ro dữ liệu chưa flush xuống đĩa. `sudo umount -R /mnt/new_root` (umount đệ quy) cũng là cách gọn hơn, thử trước khi umount tay từng dòng.

### Về ESP riêng trên ổ mới

- Cờ `esp` + `boot` trên partition ESP **bắt buộc bật** trong GParted (Manage Flags), thiếu cờ này UEFI firmware không nhận diện được partition để đưa vào boot menu.
- `grub-install` cần đúng 2 tham số `--efi-directory=/boot/efi` (điểm mount ESP trong chroot) và `--bootloader-id=<tên>` (tên entry sẽ xuất hiện trong UEFI boot menu, ví dụ `ubuntu`) — thiếu `--bootloader-id` GRUB dùng tên mặc định, khó nhận diện giữa nhiều entry.
- Xác nhận entry đã tạo đúng bằng lệnh (chạy được cả trong Live USB lẫn sau khi đã boot vào hệ điều hành mới, cần `efibootmgr`):
  ```bash
  sudo efibootmgr -v
  ```
  Phải thấy dòng `Boot0000* ubuntu` (số thứ tự có thể khác) trỏ tới `\EFI\ubuntu\shimx64.efi` trên `nvme1n1p4`.

### Về rsync

- Quên `--exclude="/home/*"` khi copy root → dữ liệu home lẫn vào root mới.
- Dấu `/` cuối đường dẫn nguồn là bắt buộc (`/mnt/oldroot/` chứ không phải `/mnt/oldroot`).
- `/swapfile` copy thẳng không cần xử lý riêng — đã xác nhận `swapon --show` hoạt động sau khi boot.

### Về fstab

- Gõ sai UUID → không boot được, cứu bằng cách boot lại Live USB, chroot lại (Bước 6), sửa lại.
- ESP mới dùng UUID dạng ngắn (`XXXX-XXXX`, ví dụ `DB34-FDB5`) vì là FAT32 — khác định dạng UUID dài của ext4, đừng nhầm lẫn khi đối chiếu `blkid`.

### An toàn dữ liệu

- Toàn bộ quy trình từ Bước 4 đến Bước 8 **không đụng tới ổ cũ** (`nvme0n1p5`, `p6` chỉ được đọc qua rsync, không ghi) — sai ở đâu, boot lại Live USB làm lại từ đó, không mất gì cho tới khi tự tay xóa ở Bước 9.

---

## 5. Link tham khảo

- [ArchWiki – GRUB (UEFI, chroot, bootloader-id)](https://wiki.archlinux.org/title/GRUB)
- [Moving Arch Linux to a new SSD with rsync – rdeeson.com](https://www.rdeeson.com/weblog/157/moving-arch-linux-to-a-new-ssd-with-rsync)
- [Migrating a Linux system from one disk to another using chroot – itstorage.net](https://itstorage.net/index.php/ldce/laa/571-migrating-a-linux-system-from-one-disk-to-another-using-chroot)
- [Backup and restore with rsync – BunsenLabs forum](https://forums.bunsenlabs.org/viewtopic.php?id=3586)
- [Increase an ext4 partition with GParted – jesusamieiro.com](https://www.jesusamieiro.com/increase-an-ext4-partition-with-gparted/)
- [How To Resize Active/Primary root Partition Using GParted – 2DayGeek](https://www.2daygeek.com/linux-resize-active-primary-root-partition-gparted/)
- [Ubuntu 20.04 LTS releases (ISO cho Live USB)](https://releases.ubuntu.com/20.04/)
- [Rufus – tạo USB boot từ Windows](https://rufus.ie)
