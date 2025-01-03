# CTF LAB

**stack frame of ctf.c**

![stack frame](./img/Lab1/stackframe_ctf.jpg)

- To transfer control from the `vuln` function to the `myfunc` function, you need to overwrite the address of the `myfucn` function on the `Return Address` of the `vuln` function.

- Create flie `flag1.txt`

![txt file](./img/lab1/image-5.png)

- Go to `gdb` to get the function address `myfunc`, the address is `0x0804851b`

![myfunc address](./img/lab1/image-1.png)

- The two addresses highlighted will be read and assigned values ​​to the two arguments `p` and `q` of the `myfuc` function respectively.We will set a breakpoint before the program checks p and q to set their values ​​to `0x4081211` and `0x44644262` respectively.

![p and q address](./img/lab1/image-3.png)

![breakpoint](./img/lab1/image-6.png)

- Run the program with the command `r $(python -c "print('a'*104 +'\x1b\x85\x04\x08')")`to transfer control to `myfunc`. Then set at position `[ebp+8]=0x04081211` and `[ebp+12]=0x44644262` to set the values ​​for $p$ and $q$. We complete the math requirements

![run](./img/lab1/image-7.png)

![complete](./img/lab1/image-9.png)
