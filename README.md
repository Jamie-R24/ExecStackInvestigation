# ExecStackInvestigation
Investigation of the NX Stack compiler defense and how to circumvent it

## Run Setup:
The files can be downloaded from the picoCTF website listed [here](https://play.picoctf.org/practice/challenge/179?page=1&search=Here%27s%20a%20). Once the files are downloaded, you are left with the executable `vuln`, `libc.so.6` and `Makefile`. 

If you try running the program with `./vuln`, you get an error because the version of libc used to compile the program is different than the host machine. You can find the correct linker file required by running `readelf -l ./vuln | grep interpreter`. Then once you know what version loader your binary wants by default, you can download the appropriate version [here](https://launchpad.net/ubuntu/bionic/amd64/libc6/2.27-3ubuntu1.4). 

Once you've downloaded the `.deb`, run `dpkg-deb -x ~/Downloads/libc6_2.27-3ubuntu1.4_amd64.deb .` to get the files in the current directory you're working in, and find the loader within `debian/lib/x86_64-linux-gnu`. Here, it's titled `ld-2.27.so`. We can copy that to our main directory with `vuln` and then run `patchelf --set-interpreter "$PWD/ld-2.27.so" vuln` and `patchelf --set-rpath "$PWD" vuln`. Now we should be able to run the file with `./vuln`. 

## Program Investigation:
Initially running the program with `./vuln`, we can see that the program spits back whatever we type in and changes the case of the letters to make it "aAaAaA". 

