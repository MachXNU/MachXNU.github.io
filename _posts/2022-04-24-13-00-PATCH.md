---
title: Patch an executable with Python
date: 2022-04-24 15:00:00 +0200
categories:  #[ TOP_CATEGORIE, SUB_CATEGORIE]
tags: []     # TAG names should always be lowercase
img_path : /2022-04-24-PATCH
---

[Hopper](https://www.hopperapp.com) is a very useful disassembler for Mac/Linux and I use it all the time when I need to reverse a binary.\
The problem is that it costs a whole 99€ to get a license and be able to export a patched binary.\
In this post, I will show how to apply patches to a binary thanks to Python.\
Note that this method can also be adapted to any programming language, such as C or others. It can also be used to automate patches if needed.

## Find something to patch 

Let's use a simple example.\
Consider the following code :
```c
#include <stdlib.h>
#include <stdio.h>

int main(){
    int age=0;
    printf("How old are you ? ");
    scanf("%d", &age);
    if (age<18){
        printf("You are a child\n");
    }
    else {
        printf("You are an adult\n");
    }
    return 0;
}
```
Assume you are given only the compiled binary, but you are only 10 y.o.\
We will patch the binary so that it displays `You are an adult` even if you are too young.

## Find what patches to apply with Hopper

Load the compiled binary in Hopper and navigate to its main function.\
It should be easy to see that there is an `if` instruction which redirects execution if you are an adult.
![Desktop View](main.png){: .shadow }
_The strings are displayed, but too far on the right to be visible here_

We must patch the jump to redirect execution to the second bloc regardless of the result of the test before.\
The solution is to replace the conditional jump `b.le` by an unconditionnal jump `b`.\
To do this, go to `Modify > Assemble instruction...` or use the keyboard shortcut `Option + A` and type the new instruction (adapt it if necessary with the correct name of the bloc) : `b loc_100003f18`\
![Desktop View](assemble.png){: .shadow width="500"}\
As you can see after patching, the "child" region is never executed, and the execution flow will directly jump to the "adult" bloc.\
![Desktop View](jump.png){: .shadow width="500"}\
Now, the first thing you need to note down is the **offset** from the beginning of your binary.\
It is the offset from the binary's base `0x100000000` to your instruction.\
To double check what the base is, navigate to the Mach-O header :\
![Desktop View](header.png){: .shadow width="500"}
_This is the Mach-O header_
For us, the offset will be `0x100003f04 - 0x100000000 = 0x3f04`\
Next, we need to find what the new instruction is like in binary code.\
Select the instruction in the ASM view and switch to hexadecimal view.\
![Desktop View](gotohex.png){: .shadow}
_Make sure you have selected the patched instruction_ 
You will see values in red, these are the new hex values for the patched instruction.\
![Desktop View](hex.png){: .shadow}
_The patched instruction has its hex representation in red_
Note them down as well, as we will need them later on.\
The Hopper part is done, so make sure you got the **offset** and the **hex** representation of the instruction.

## Write a Python script and apply patches

I have provided the script, just after this text.\
As you can see, we are first loading the original binary in binary mode. This gives us a `<bytes>` object. Since `<bytes>` objects are immutable, we need to convert it to a (mutable) array.\
Once this is done, we simply replace the value at the **offset** by the new **hex** representation.\
That is why we needed both values.\
Here is the script :
```python
#!/usr/bin/python3

import sys
 
if len(sys.argv) != 2:
    print('Please input path to the binary you want to patch.')
    sys.exit()

def replace(content, offset, instruction):
    print("[i] Replacing instruction at offset", hex(offset))
    content[offset : offset+len(instruction)] = instruction

def bytes_to_array(input):
    n = len(input)
    L = [b'\x00']*n
    for i in range(n):
        L[i] = input[i]
    return L

orig_file = sys.argv[1]
patched_file = orig_file + '_patched'
file = open(orig_file, 'rb')
content = bytes_to_array(file.read())
file.close()
replace(content, 0x3f04, b'\x05\x00\x00\x14')  # replace if necessary
file = open(patched_file, 'wb')
file.write(bytes(content))
file.close()
print("Patched file created")
```
Then, patch the executable :
```bash
./patcher.py <path-to-executable>
```
The produced binary is called `<original_name_patched>` and can be found at the same path than the original file.\
What you have to do next is configure your system to execute this patched binary. Many systems include protections against this type of attack, and iOS or macOS are no exceptions to this.\
See [my other blog article about running patched executables on macOS](https://machxnu.github.io/posts/15-00-MAC-CODESIGN/) to effectively run this patched executable on your machine ! \
However, as a spoiler, I can already show you that our patch worked :
![Desktop View](patched.png){: .shadow}
_Wow, society is moving at a fast pace !_

## Conclusion
To summarize, you can use Hopper to see what/where patches should be applied and see what the new instructions will look like after the patch.\
Then, a simple script (which can of course be adapted to any language) applies the patches and produces the patched executable.\
Finally, you have to configure your system to run this patched executable, but this is too OS-specific to be discussed here. 
