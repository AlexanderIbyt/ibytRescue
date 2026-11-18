# Rescue

A small data recovery tool written in ibyt. Boot a PC from a stick, point Rescue at a damaged or reformatted drive, and get your photos, PDFs and Office documents back onto a second drive.

Rescue runs on bare metal (BIOS boot, no operating system). It reads the drive directly through the BIOS and writes the recovered files to a different drive that you choose.

## What it does

Rescue has two ways to find files:

**1. Deleted files on a FAT drive**
Walks through all folders of FAT12, FAT16 and FAT32 partitions and lists entries that are marked as deleted. File names are kept; only the first letter is lost (shown as `_`). The content is read from the first cluster onwards as one contiguous run.

**2. Carving (search the whole drive)**
Scans the drive sector by sector for the beginning of known file types and finds the end inside the data. This works even if the drive was quick-formatted or the folder structure is gone, but the original names are lost.

| Type | How the end is found |
|---|---|
| JPEG | walks the segments of the file, so an embedded EXIF thumbnail does not cut the photo short |
| PNG | `IEND` chunk |
| PDF | first `%%EOF` |
| ZIP / Office (DOCX, XLSX, ...) | End-of-Central-Directory record, checked for plausibility |

## Safety

- **The source drive is never written.** The only writes go through the FAT writer to the target drive.
- Source and target must be different drives. This is checked before every save.
- Recovered files are stored in a folder `/RESCUE/` on the first FAT partition of the target drive.

Always recover to a **different physical drive** than the one you are rescuing, and never install or save anything onto the damaged drive first.

## Using it

1. Set up the Rescue stick with ibytWriter (see Building) and boot from it (BIOS/Legacy/CSM mode, Secure Boot off).
2. Choose the **source** drive and then the **target** drive (key `0` to `7` = BIOS drive number). Esc goes back.
3. Pick a method:
   - `1` deleted files on a FAT drive
   - `2` search the whole drive for photos, PDF and Office files
   - `Esc` choose other drives
4. A hit list appears (15 rows per page):

   | Key | Action |
   |---|---|
   | Up / Down, Page Up / Page Down | move |
   | Enter | recover the selected hit |
   | `A` | recover all hits |
   | Esc | back |

5. Files are saved as `0001__ATEI_20.TXT` (deleted FAT file, first letter lost) or `0002_file.jpg` (carved file). Large files take a while; Rescue shows "Reading, please wait..." while it works. Esc stops a running search.

## Building

You need the ibyt compiler, NASM and the ibyt `library` folder next to the source.

```
cd rescue
<ibyt-compiler>          # produces MyProgram.img
```

Test in an emulator with three drives (Rescue stick, source, target):

```
qemu-system-x86_64 \
  -drive format=raw,file=MyProgram.img \
  -drive format=raw,file=source.img \
  -drive format=raw,file=target.img
```

Write the image to a stick with **ibytWriter**, the ibyt in-house tool. It lists the USB drives with numbers, asks "Which drive?" and asks for confirmation before it writes. This turns the stick into a Rescue start USB. Or do it by hand (`/dev/sdX` is the whole stick, **not your hard disk**):

```
sudo dd if=MyProgram.img of=/dev/sdX bs=1M conv=fsync status=progress
```

## Tested

In QEMU with FAT12, FAT16 and FAT32 images:

- FAT32 with three deleted files in three folders: all found and recovered **byte-identical**; source drive unchanged afterwards (checked with `cmp`).
- Quick-formatted drive: carving found 5 of 5 files (2 JPEG, PNG, PDF, DOCX), all byte-identical.
- FAT16 with 40 deleted files: paging and selection work. FAT12 (floppy size) works.

**Not tested yet:** drives larger than 2 TB.

## Limits (version 1)

- Only **contiguous** files are recovered. Fragmented files are cut off or wrong.
- Long file names (VFAT) are not read; the first letter of a deleted name is lost.
- A PDF with several `%%EOF` markers (linearized or incrementally updated) is cut at the first one.
- A file can be at most 32 MiB; the carving search window is 8 MiB.
- No NTFS, exFAT or encrypted drives (for example BitLocker).
- Reading goes through the BIOS (INT 13h) and is slow. A whole large drive can take hours.

## No warranty

Data recovery is never guaranteed. Rescue is provided as it is, without any warranty (see `LICENSE`). If the data is valuable, stop using the drive and consider a professional service.

## License

Rescue is released under the **MIT License**, see the file `LICENSE`. You may use, change and share it freely, also commercially, as long as the copyright notice and the license text are kept.

Copyright (c) 2026 Alexander Zima, https://www.ibyt.tech
