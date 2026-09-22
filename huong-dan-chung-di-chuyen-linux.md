# Hướng dẫn chung: Chuyển/nhân bản Linux sang ổ cứng khác

> Tài liệu tham khảo tổng quát — áp dụng cho mọi máy dual-boot Windows + Linux (Ubuntu/Debian và
> họ hàng dùng GRUB2) muốn chuyển hệ thống Linux hiện tại sang ổ mới, hoặc cài thêm một bản Linux
> thứ hai song song. Không gắn với thông số của máy cụ thể nào — thay các giá trị trong `<...>`
> bằng số liệu thật của máy bạn trước khi chạy lệnh.

---

## 0. Hai kịch bản — xác định trước khi bắt đầu

| Kịch bản | Khi nào chọn | Độ phức tạp |
|---|---|---|
| **A. Di chuyển (migrate)** | Đã có Linux đang chạy, muốn chuyển nguyên trạng (config, phần mềm đã cài, dữ liệu) sang ổ khác | Trung bình — clone + sửa boot |
| **B. Cài mới song song (dual-install)** | Muốn có thêm 1 bản Linux sạch, giữ nguyên bản cũ, chọn lúc boot | Thấp — cài như bình thường, GRUB tự thêm entry |

Tài liệu này tập trung vào **kịch bản A** (phức tạp hơn, dễ sai hơn); kịch bản B ghi ở [Phụ lục B](#phụ-lục-b--cài-thêm-linux-song-song-kịch-bản-b).

---

## 1. Kiểm tra trước khi làm

Chạy trên hệ thống Linux đang dùng (chưa cần boot Live USB):

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT,UUID
sudo parted -l
df -h
cat /etc/fstab
swapon --show
lsb_release -d          # xác định đúng version đang cài, để tải Live USB cùng bản
```

Ghi lại các thông tin sau — sẽ dùng xuyên suốt tài liệu:

- `<OLD_ROOT>` = thiết bị partition root hiện tại (vd `/dev/nvme0n1p5`)
- `<OLD_HOME>` = thiết bị partition home hiện tại (nếu tách riêng)
- `<OLD_ESP>` = thiết bị ESP hiện tại (vd `/dev/nvme0n1p2`)
- `<OLD_ROOT_UUID>`, `<OLD_HOME_UUID>` = UUID tương ứng, lấy từ `blkid` hoặc `lsblk -f`
- `<NEW_DISK>` = tên thiết bị ổ đích, cả ổ (vd `/dev/nvme1n1`, `/dev/sdb`) — **không phải partition**
- `<DISTRO_VERSION>` = phiên bản Linux đang chạy

### Đơn vị GB vs GiB — đọc trước khi tính dung lượng

Nhà sản xuất ghi dung lượng theo **GB thập phân** (1GB = 10⁹ byte); Linux/GParted hiển thị theo
**GiB nhị phân** (1GiB = 2³⁰ byte). Ổ ghi "500GB" thực tế chỉ có **~465,8 GiB** khả dụng. Luôn
chia partition theo con số GParted hiển thị thật, đừng cộng trừ theo nhãn "GB" trên hộp ổ cứng —
chênh lệch ~7% dễ khiến partition cuối cùng thiếu vài GB hoặc lỗi "not enough space".

---

## 2. Chuẩn bị Live USB

1. Tải ISO **đúng phiên bản** đang chạy (`<DISTRO_VERSION>`) — cùng version giúp `grub-install`/
   `update-grub` chạy trong chroot tương thích với hệ thống đích.
   - Ubuntu: https://releases.ubuntu.com/
   - Debian: https://www.debian.org/distrib/
2. Ghi ISO ra USB (≥ 8GB) bằng [Rufus](https://rufus.ie) (Windows) hoặc Startup Disk Creator/
   `dd` (Linux).
3. Boot lại máy, vào Boot Menu (phím tùy mainboard: F12/F11/ESC/Del — thử lần lượt nếu không chắc).
4. Chọn dòng có chữ **UEFI: <tên USB>**. Bỏ qua dòng không có "UEFI" nếu ổ đang dùng GPT (chọn
   Legacy trên ổ GPT gây lỗi boot).
5. Ở màn hình cài đặt, chọn **"Try Ubuntu"** / **"Try or Install"** — không chọn cài đặt.
   Live session không mount ổ hệ thống, an toàn để thao tác partition.

⚠️ **Không thực hiện resize/clone khi đang chạy trực tiếp trên hệ điều hành đó** — filesystem
đang mở, ghi log, có tiến trình dùng file → dữ liệu dễ không nhất quán giữa lúc copy.

---

## 3. Tạo partition trên ổ mới bằng GParted

GParted có sẵn trong hầu hết Live USB Ubuntu/Debian. Mở bằng `sudo gparted` hoặc qua menu; thiếu
thì `sudo apt install gparted`.

**Quyết định trước: ESP dùng chung hay ESP riêng?**

| Phương án | Ưu điểm | Nhược điểm |
|---|---|---|
| Dùng chung ESP với ổ cũ | Đơn giản, ít bước hơn | **Ổ mới không boot độc lập** — rút ổ cũ ra là không boot được |
| Tạo ESP riêng trên ổ mới | Ổ mới boot độc lập hoàn toàn, có thể rút ổ cũ | Thêm 1 partition, thêm bước gắn cờ + cài GRUB đúng target |

Khuyến nghị **luôn tạo ESP riêng** trừ khi biết chắc sẽ không bao giờ tháo ổ cũ ra khỏi máy.

**Các bước trong GParted:**

1. Chọn đúng ổ đích ở dropdown góc trên phải (**`<NEW_DISK>`**).
   > ⚠️ Chọn nhầm ổ đang chạy hệ điều hành là mất toàn bộ dữ liệu ổ đó ngay khi Apply. Phân biệt
   > bằng: ổ đích thường chỉ có vùng "unallocated" lớn, không có partition ext4/ntfs đang dùng.
2. Nếu ổ đích đang có partition cũ (từ Windows, hoặc lần cài trước) cần xóa: chuột phải từng
   partition → **Delete**.
3. Tạo partition **root**: chuột phải vùng unallocated → **New** → filesystem `ext4`, nhập size
   theo MiB (vd `102400` MiB ≈ 100 GiB) → **Add**.
4. (Nếu tách home riêng) Tạo partition **home**: tương tự, filesystem `ext4`, để trống "New size"
   dùng mặc định nếu muốn lấy hết phần còn lại, hoặc trừ lại dung lượng cho ESP nếu làm ở bước 5.
5. (Nếu chọn ESP riêng) Tạo partition **ESP**: filesystem `fat32`, size `512`–`1024` MiB. Sau khi
   tạo, chuột phải partition này → **Manage Flags** → tick **`esp`** (tự động kèm `boot`).
   > ⚠️ Thiếu cờ `esp` thì UEFI firmware không nhận diện được partition này để boot.
6. Bấm **✓ (Apply All Operations)**, chờ hoàn tất.
7. Xác nhận bằng:
   ```bash
   lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT <NEW_DISK>
   ```
   > Lưu ý cột là `MOUNTPOINT` (số ít) — một số bản `lsblk` trong live image báo lỗi
   > `unknown column: MOUNTPOINTS` nếu gõ nhầm số nhiều.

Ghi lại tên partition thật vừa tạo — **không giả định thứ tự**, luôn đọc từ `lsblk`:

- `<NEW_ROOT>` (vd `/dev/nvme1n1p2`)
- `<NEW_HOME>` (nếu có, vd `/dev/nvme1n1p3`)
- `<NEW_ESP>` (nếu tạo riêng, vd `/dev/nvme1n1p4`)

---

## 4. Mount nguồn và đích

```bash
sudo mkdir -p /mnt/oldroot /mnt/new_root
sudo mount <OLD_ROOT> /mnt/oldroot
sudo mount <NEW_ROOT> /mnt/new_root

# Nếu home tách riêng:
sudo mkdir -p /mnt/oldhome
sudo mount <OLD_HOME> /mnt/oldhome
sudo mkdir -p /mnt/new_root/home
sudo mount <NEW_HOME> /mnt/new_root/home

# Nếu tạo ESP riêng:
sudo mkdir -p /mnt/new_root/boot/efi
sudo mount <NEW_ESP> /mnt/new_root/boot/efi
```

Kiểm tra đã mount đúng trước khi copy:

```bash
df -h /mnt/oldroot /mnt/oldhome /mnt/new_root /mnt/new_root/home 2>/dev/null
```

---

## 5. Copy dữ liệu bằng rsync

```bash
sudo rsync -aAXH --info=progress2 /mnt/oldroot/ /mnt/new_root/ \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/*","/boot/efi/*"}
```

Nếu home tách riêng, copy thêm:

```bash
sudo rsync -aAXH --info=progress2 /mnt/oldhome/ /mnt/new_root/home/
```

**Giải thích cờ:** `-a` giữ permission/owner/timestamp/symlink, `-A` ACL, `-X` extended
attributes, `-H` hard link, `--info=progress2` hiện % tiến độ tổng.

**Ghi chú về swapfile:** nếu hệ thống dùng file `/swapfile` thay vì swap partition, **không cần
exclude riêng** — rsync copy nó như file thường và `swapon` hoạt động lại bình thường sau khi
boot (đã xác nhận qua thực tế). Không cần `fallocate`/`mkswap` tạo lại trừ khi muốn đổi kích
thước swap.

**Kiểm tra sau copy:**

```bash
sudo du -sh /mnt/oldroot /mnt/new_root
sudo du -sh /mnt/oldhome /mnt/new_root/home 2>/dev/null
```

Muốn xem trước không ghi gì thật: thêm cờ `-n` (dry-run) vào lệnh rsync.

> **Lưu ý dấu `/` cuối đường dẫn nguồn là bắt buộc** (`/mnt/oldroot/` chứ không phải
> `/mnt/oldroot`) — thiếu nó rsync tạo thêm thư mục lồng thay vì copy nội dung vào thẳng đích.

---

## 6. Chroot: sửa fstab, cài lại GRUB

### 6.1 Mount hệ thống ảo và chroot

```bash
sudo mount --bind /dev      /mnt/new_root/dev
sudo mount --bind /dev/pts  /mnt/new_root/dev/pts
sudo mount --bind /proc     /mnt/new_root/proc
sudo mount --bind /sys      /mnt/new_root/sys
sudo chroot /mnt/new_root
```

Dấu nhắc đổi thành `root@...:/#` là đã vào chroot.

### 6.2 Lấy UUID mới

```bash
blkid
```

Ghi lại UUID của `<NEW_ROOT>`, `<NEW_HOME>` (nếu có), `<NEW_ESP>` (nếu có — filesystem FAT32 có
UUID dạng ngắn `XXXX-XXXX`, khác định dạng UUID dài của ext4, đừng nhầm khi đối chiếu).

### 6.3 Sửa `/etc/fstab`

```bash
nano /etc/fstab
```

Thay UUID cũ bằng UUID mới cho từng dòng tương ứng:

| Dòng mount | UUID cũ | Thay bằng |
|---|---|---|
| `/` | `<OLD_ROOT_UUID>` | UUID của `<NEW_ROOT>` |
| `/home` (nếu tách riêng) | `<OLD_HOME_UUID>` | UUID của `<NEW_HOME>` |
| `/boot/efi` (chỉ nếu tạo ESP riêng) | UUID ESP cũ | UUID của `<NEW_ESP>` |

Dòng `/boot/efi` **giữ nguyên** nếu chọn phương án dùng chung ESP ổ cũ. Dòng khai báo swap (vd
`/swapfile ... swap`) giữ nguyên không đổi trong mọi trường hợp.

> 🔴 Gõ sai một ký tự UUID khiến hệ thống không boot được (rơi vào `(initramfs)` hoặc emergency
> mode). Luôn đối chiếu `blkid` trước khi gõ, không đoán/gõ tắt.

### 6.4 Cập nhật GRUB

**Nếu dùng chung ESP với ổ cũ** (đã mount `<OLD_ESP>` vào `/mnt/new_root/boot/efi` trước khi
chroot):

```bash
update-grub
grub-install <NEW_DISK>
```

**Nếu tạo ESP riêng trên ổ mới:**

```bash
update-grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=<TÊN_ENTRY> <NEW_DISK>
```

`<TÊN_ENTRY>` là tên hiển thị trong UEFI boot menu (vd `ubuntu`) — luôn đặt `--bootloader-id` rõ
ràng, thiếu nó GRUB dùng tên mặc định khó phân biệt giữa nhiều entry.

```bash
exit
```

Xác nhận entry UEFI đã tạo đúng (chạy được cả trong chroot lẫn sau khi đã boot vào hệ thống mới):

```bash
efibootmgr -v
```

Phải thấy entry mới trỏ đúng `\EFI\<TÊN_ENTRY>\...efi` trên `<NEW_ESP>` (hoặc `<OLD_ESP>` nếu
dùng chung).

---

## 7. Unmount và reboot

**Không umount thẳng `/mnt/new_root`** — sẽ báo lỗi `target is busy` vì còn mount con bên trong
(`dev`, `proc`, `sys`, và `efivarfs` — tự sinh khi chroot đụng tới `/sys/firmware/efi/efivars`,
ví dụ lúc chạy `grub-install`).

**Cách nhanh — umount đệ quy:**

```bash
sudo umount -R /mnt/new_root
sudo umount /mnt/oldroot /mnt/oldhome 2>/dev/null
```

**Nếu vẫn báo busy**, umount thủ công theo thứ tự từ sâu nhất ra ngoài:

```bash
sudo umount /mnt/new_root/sys/firmware/efi/efivars
sudo umount /mnt/new_root/sys
sudo umount /mnt/new_root/proc
sudo umount /mnt/new_root/dev/pts
sudo umount /mnt/new_root/dev
sudo umount /mnt/new_root/boot/efi
sudo umount /mnt/new_root/home
sudo umount /mnt/new_root
```

Kiểm tra sạch trước khi reboot:

```bash
mount | grep '/mnt/new_root'     # không được ra dòng nào
```

> ⚠️ Nếu vẫn còn dòng nào, **dừng lại, xử lý tiếp — đừng dùng `umount -f`/`-l` để né lỗi và đừng
> reboot ngay.** Force-umount có thể bỏ sót dữ liệu chưa flush xuống đĩa.

```bash
sudo reboot
```

Rút USB lúc máy khởi động lại, chọn entry Linux mới ở boot menu/GRUB.

---

## 8. Xác nhận sau khi boot

```bash
findmnt /
findmnt /home
findmnt /boot/efi
lsblk -f
swapon --show
df -h
```

Đối chiếu: `/` → `<NEW_ROOT>`, `/home` → `<NEW_HOME>` (nếu tách riêng), `/boot/efi` → `<NEW_ESP>`
hoặc `<OLD_ESP>` tùy phương án đã chọn.

Nếu dual-boot với Windows: reboot thêm lần nữa, vào boot menu chọn **Windows Boot Manager** để
xác nhận Windows vẫn boot bình thường — việc chuyển Linux không ảnh hưởng tới ESP của Windows nếu
đã chọn phương án ESP riêng.

---

## 9. Dọn ổ cũ (không vội, làm sau khi đã xác nhận ổn định)

Giữ `<OLD_ROOT>`/`<OLD_HOME>` nguyên vẹn **ít nhất vài ngày đến 1 tuần** làm bản dự phòng trước
khi:

- Xóa/format lấy lại dung lượng bằng GParted, hoặc
- Mở rộng partition Windows liền kề bằng Disk Management (chỉ được nếu vùng trống nằm ngay sau
  partition đó).
- Chạy lại `sudo update-grub` sau khi xóa để dọn entry cũ khỏi menu GRUB.

Toàn bộ quy trình từ mục 4 đến 8 **không ghi đè lên ổ cũ** — chỉ đọc qua rsync. Sai ở đâu, boot
lại Live USB làm lại từ đó, không mất gì cho tới khi tự tay xóa ở bước này.

---

## Phụ lục A — Bảng lỗi thường gặp

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `unknown column: MOUNTPOINTS` | Gõ nhầm số nhiều cho cột `lsblk` | Dùng `MOUNTPOINT` (số ít) |
| `target is busy` khi umount | Còn mount con (`dev`/`proc`/`sys`/`efivarfs`) bên trong | Umount đệ quy `-R`, hoặc tháo tay theo thứ tự từ trong ra ngoài (mục 7) |
| Boot vào `(initramfs)` / emergency mode | UUID sai trong `/etc/fstab` | Boot lại Live USB, chroot lại, sửa `/etc/fstab` theo `blkid` thật |
| GRUB không thấy Windows | `os-prober` không chạy hoặc bị tắt | Trong chroot: `os-prober` rồi `update-grub`; kiểm tra `/etc/default/grub` không có `GRUB_DISABLE_OS_PROBER=true` |
| UEFI không hiện entry mới | Thiếu cờ `esp` trên partition ESP, hoặc thiếu `--bootloader-id` | GParted → Manage Flags → tick `esp`; chạy lại `grub-install` với `--bootloader-id` rõ ràng |
| rsync tạo thư mục lồng (`new_root/oldroot/`) | Thiếu dấu `/` cuối đường dẫn nguồn | Luôn thêm `/` sau thư mục nguồn: `/mnt/oldroot/` |
| Home cũ lẫn vào root mới | Quên `--exclude="/home/*"` khi copy root | Kiểm tra kỹ câu lệnh rsync trước khi Enter, hoặc `du -sh` đối chiếu dung lượng sau copy |

---

## Phụ lục B — Cài thêm Linux song song (kịch bản B)

Nếu chỉ muốn thêm 1 bản Linux mới (không di chuyển bản cũ):

1. Không cần Live USB riêng cho việc di chuyển — dùng ngay USB cài đặt bản Linux muốn cài.
2. Trong bước chọn ổ đích lúc cài đặt (installer), chọn **ổ mới** (`<NEW_DISK>`), chọn
   **"Something else" / Manual partitioning** (Ubuntu) để tự chia root/home/swap thay vì để
   installer tự động ghi đè ổ khác.
3. Với UEFI: **dùng chung ESP hiện có** (installer thường tự phát hiện ESP đã có sẵn và hỏi có
   dùng lại không) — không cần tạo ESP mới trừ khi muốn bản mới boot độc lập.
4. Cài xong, GRUB (`os-prober`) tự dò các hệ điều hành khác trên máy (Windows, bản Linux cũ) và
   thêm vào boot menu. Nếu thiếu entry, chạy trong bản mới:
   ```bash
   sudo apt install os-prober
   sudo os-prober
   sudo update-grub
   ```
5. Không giới hạn số bản Linux cài song song — chỉ giới hạn bởi dung lượng đĩa và số partition.

---

## Link tham khảo

- [ArchWiki – GRUB (UEFI, chroot, bootloader-id)](https://wiki.archlinux.org/title/GRUB)
- [ArchWiki – Chroot (mount hệ thống ảo, sửa hệ thống từ live)](https://wiki.archlinux.org/title/Chroot)
- [Moving Arch Linux to a new SSD with rsync – rdeeson.com](https://www.rdeeson.com/weblog/157/moving-arch-linux-to-a-new-ssd-with-rsync)
- [Migrating a Linux system from one disk to another using chroot – itstorage.net](https://itstorage.net/index.php/ldce/laa/571-migrating-a-linux-system-from-one-disk-to-another-using-chroot)
- [Backup and restore with rsync – BunsenLabs forum](https://forums.bunsenlabs.org/viewtopic.php?id=3586)
- [Increase an ext4 partition with GParted – jesusamieiro.com](https://www.jesusamieiro.com/increase-an-ext4-partition-with-gparted/)
- [How To Resize Active/Primary root Partition Using GParted – 2DayGeek](https://www.2daygeek.com/linux-resize-active-primary-root-partition-gparted/)
- [Ubuntu releases (ISO Live USB)](https://releases.ubuntu.com/)
- [Rufus – tạo USB boot từ Windows](https://rufus.ie)
