# CTF LAB

**stack frame of ctf.c**

![stack frame](./img/Ảnh%20chụp%20màn%20hình%202024-09-10%20203523.png)

- To transfer control from the `vuln` function to the `myfunc` function, you need to overwrite the address of the `myfucn` function on the `Return Address` of the `vuln` function.

- Create flie `flag1.txt`

![txt file](/img/image-5.png)

- Go to `gdb` to get the function address `myfunc`, the address is `0x0804851b`

![myfunc address](./img/image-1.png)

- The two addresses highlighted will be read and assigned values ​​to the two arguments `p` and `q` of the `myfuc` function respectively.We will set a breakpoint before the program checks p and q to set their values ​​to `0x4081211` and `0x44644262` respectively.

![p and q address](./img/image-3.png)

![breakpoint](./img/image-6.png)

- Run the program with the command `r $(python -c "print('a'*104 +'\x1b\x85\x04\x08')")`to transfer control to `myfunc`. Then set at position `[ebp+8]=0x04081211` and `[ebp+12]=0x44644262` to set the values ​​for $p$ and $q$. We complete the math requirements

![run](./img/image-7.png)

![complete](./img/image-9.png)
