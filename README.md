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


Although we now know the offset, we can't just override the return address because of the security feature, `NX Enabled`. This disables any code from being executed from the stack, so a simple return address overwrite won't work. If we point our overwritten return address to a function from the standard libc library, we can leverage the power of those functions. The standard libc libary is located in an always-executable location, so we would have no problem pointing to one of those functions. 
