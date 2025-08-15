# ExecStackInvestigation
Investigation of the NX Stack compiler defense and how to circumvent it

## Run Setup:
The files can be downloaded from the picoCTF website listed [here](https://play.picoctf.org/practice/challenge/179?page=1&search=Here%27s%20a%20). Once the files are downloaded, you are left with the executable `vuln`, `libc.so.6` and `Makefile`. 

If you try running the program with `./vuln`, you get an error because the version of libc used to compile the program is different than the host machine. You can find the correct linker file required by running `readelf -l ./vuln | grep interpreter`. Then once you know what version loader your binary wants by default, you can download the appropriate version [here](https://launchpad.net/ubuntu/bionic/amd64/libc6/2.27-3ubuntu1.4). 

Once you've downloaded the `.deb`, run `dpkg-deb -x ~/Downloads/libc6_2.27-3ubuntu1.4_amd64.deb .` to get the files in the current directory you're working in, and find the loader within `debian/lib/x86_64-linux-gnu`. Here, it's titled `ld-2.27.so`. We can copy that to our main directory with `vuln` and then run `patchelf --set-interpreter "$PWD/ld-2.27.so" vuln` and `patchelf --set-rpath "$PWD" vuln`. Now we should be able to run the file with `./vuln`. 

## Program Investigation:
Let's run `checksec` on the executable to see what defenses the program was compiled with. 

<img width="3578" height="260" alt="image" src="https://github.com/user-attachments/assets/1236a3d7-53ce-45eb-8c7a-6ee2394e51d3" />


We can see that `NX Enabled` means that the stack is marked as non-executable. This is important for later. 

Initially running the program with `./vuln`, we can see that the program spits back whatever we type in and changes the case of the letters to make it "aAaAaA". 

<img width="3840" height="2400" alt="image" src="https://github.com/user-attachments/assets/da25ee31-1380-494a-8d84-098c69e34941" />


We can open the file in ghidra to see the source code and potentially find a vulnerability. We can see the `main()` function is defined as such:

<img width="1828" height="1904" alt="image" src="https://github.com/user-attachments/assets/83afe844-bba1-4072-b1c7-83e825025d56" />


Investigating further, we can see the function at the bottom of the `main()` function titled `do_stuff()`. This function is in an infinite `do while` loop, indicating it's responsible for the "echoing" of the program. 

<img width="1468" height="1104" alt="image" src="https://github.com/user-attachments/assets/ba7cd2a5-00a8-491f-8930-6a2c05206679" />


We can see that the `scanf()` function call here terminates at the appearance of a `"\n"` character. This is susceptible to a buffer overflow as whatever is entered by the user up to a newline, gets put into the buffer without any bounds checking. 

Let's throw the exectuable into `gdb` and find where we override the `rip` register. We can use patterns to do this. 

<img width="3840" height="2400" alt="image" src="https://github.com/user-attachments/assets/e0828527-557b-4eb2-96b2-3a93140f38a8" />

<img width="3840" height="2400" alt="image" src="https://github.com/user-attachments/assets/0606ec04-3703-464a-bf4a-bb078972e1e5" />


Although we now know the offset, we can't just override the return address because of the security feature, `NX Enabled`. This disables any code from being executed from the stack, so a simple return address overwrite won't work. Also, there are no methods within the binary that easily allow us to obtain the `flag.txt` file. We need a way to access the file without necessarily printing it. 

If we point our overwritten return address to a function from the standard libc library, we can leverage the power of those functions. The standard libc libary is located in an always-executable location, so we would have no problem pointing to one of those functions. We need to find where the libc libary was loaded during runtime. Then, we can use `system()` in combination with `/bin/sh` to pop a shell and obtain the flag.

We can use the `puts()` function at the end of the `do_stuff()` function to leak the address of the libc function to `stdout`. I'm choosing `setbuff()` as the libc function. Let's use ROPGadget to help us. We can run `ROPgadget --binary vuln | grep "pop rdi"`. This gives us the ability to modify rdi.

<img width="1172" height="238" alt="image" src="https://github.com/user-attachments/assets/8ddcc21e-7975-4906-b1a3-58ccca3ba8d4" />


We can use ghidra to find the address of `puts()` and `setbuf()`.

<img width="2516" height="900" alt="image" src="https://github.com/user-attachments/assets/0a8f4bdc-7f20-4ec6-81ac-833b98a00c84" />

<img width="2264" height="236" alt="image" src="https://github.com/user-attachments/assets/44676f7c-3c11-47bd-99c1-55b3d1e9345f" />


The address for `setbuf()` is from the GOT (global offset table). Referrencing the GOT entry for `setbuf()` should return us the actual memory location of the function. 

We have the necessary addresses now so we can leak the actual address of `setbuf()` within the program. We can throw together this initial python script to obtain the information. 

<img width="3840" height="2400" alt="image" src="https://github.com/user-attachments/assets/bd6bee38-74eb-412a-9fc2-5871a93bc479" />


Running this script with `python3 <name-of-script>` results in the following output:

<img width="1192" height="324" alt="image" src="https://github.com/user-attachments/assets/20ad4d52-490a-4bcd-b1b9-9bce2eb0266f" />


With this info we can calculate where libc is located in memory. Using the `libc.so.6` file provided, we can calculate the offset of the start of libc to `setbuf()`. 

<img width="1926" height="230" alt="image" src="https://github.com/user-attachments/assets/3306ca0a-acfa-434d-a3f4-df687648856d" />


The offset is `0x88540`. We can get the base address of libc by doing simple hex calculation. `libc_start = leaked_setbuf_address - offset`. Let's add this to the python script.

<img width="1430" height="478" alt="image" src="https://github.com/user-attachments/assets/cd233d5e-8c29-4ae9-a993-3c82fd893628" />


Now we can find the address of the `system()` function within libc. By using `system()`, we can pass in `/bin/sh` and launch a shell to capture our flag. We can also find the address of `/bin/sh` within the libc address space. 

<img width="1940" height="176" alt="image" src="https://github.com/user-attachments/assets/2adadbd7-bd2d-4981-8260-9301bda6a514" />

<img width="1340" height="368" alt="image" src="https://github.com/user-attachments/assets/b5635fe6-a86e-4346-ab25-190c422a1238" />

<img width="986" height="168" alt="image" src="https://github.com/user-attachments/assets/652ab9fc-d1af-4268-b7bf-fdc6420a9eed" />

<img width="1360" height="490" alt="image" src="https://github.com/user-attachments/assets/3a4af4d8-6a48-4c05-ad4c-6f5ceeacac8a" />


Now we can craft our final payload. But first, let's find the address for an ROPgadget ret so that we can make sure the stack is aligned so no segmentation fault occurs. Then we can incorporate this into the final payload. That is as simple as running `ROPgadget --binary vuln | grep "ret"`. I picked the address that just showed ret and nothing else which happened to be `0x40052e`.

<img width="1386" height="1134" alt="image" src="https://github.com/user-attachments/assets/cd648a85-effb-4f71-8f55-e7956a3eff65" />


Running this locally pops us a shell and completes our exploit.

<img width="3788" height="870" alt="image" src="https://github.com/user-attachments/assets/4b457936-278b-4392-aec5-a1257b74fa74" />

We can modify the script to run remotely on the picoCTF server by adding these lines at the top instead of `p.process("./vuln")`.
```
p = remote('mercury.picoctf.net', 49464)  
p.recvuntil(b"WeLcOmE To mY EcHo sErVeR!")
```

Running this (named `remote_exploit.py` in the Github), pops a shell on the server where we can use `cat flag.txt` to get the flag. 

<img width="2978" height="944" alt="image" src="https://github.com/user-attachments/assets/95a6edbc-8890-46aa-844d-1db03be1ee1e" />





