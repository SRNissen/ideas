Consider a program that writes to adjacent bytes:

    ////////////////////////////////////////////////////////////////////////////////
    #include <cstdlib>
        
    int main(int arg, char**)
    {
            auto size = 1'0000'0000z;
            auto buffer = (char*)std::malloc(size); // malloc should rarely be used
                                                    // in real programs, it's just
                                                    // here for the example.
            
         // Example with 1 byte
            buffer[0] = 'a';
            
         // Example with 2 bytes
            buffer[64] = 'b';
            buffer[65] = 'c';
            
         // Example with 3 bytes
            buffer[128] = 'd';
            buffer[129] = 'e';
            buffer[130] = 'f';
            
         // Example with 4 bytes
            buffer[256] = 'g';
            buffer[257] = 'h';
            buffer[258] = 'i';
            buffer[259] = 'j';
            
         // Example with 8 bytes
            buffer[512] = 'k';
            buffer[513] = 'l';
            buffer[514] = 'm';
            buffer[515] = 'n';
            buffer[516] = 'o';
            buffer[517] = 'p';
            buffer[518] = 'q';
            buffer[519] = 'r';
            
         // Example with 4 bytes with 1-byte padding in between
            buffer[1024] = 's';
            // skip 1025
            buffer[1026] = 't';
            // skip 1027
            buffer[1028] = 'u';
            // skip 1029
            buffer[1030] = 'v';
            // skip 1031
        
            return buffer[arg];
    }
    ////////////////////////////////////////////////////////////////////////////////

Notice the access patterns here - I've spaced the indexes out so each example doesn't step on the other ones, but they're all in the same memory buffer.

Compiled, it turns into https://godbolt.org/z/h6EEKch75

This probably means nothing to you at this stage in your journey, but let me pull out the parts that are relevant to your question, which show what actually gets copied around (`rax` is the pointer to the buffer)

    // 1 byte
    mov     byte ptr [rax], 97 // 'a'
    

    // 2 bytes
    mov     word ptr [rax + 64], 25442 // 'bc'
    

    // 3 bytes
    mov     word ptr [rax + 128], 25956 // 'de'
    mov     byte ptr [rax + 130], 102   // 'f'
    

    // 4 bytes
    mov     dword ptr [rax + 256], 1785292903 // 'ghij'
    

    // 8 bytes
    movabs  rcx, 8246496016588434539 // 'klmnopqr'
    mov     qword ptr [rax + 512], rcx
    

    // 4 padded bytes
    mov     byte ptr [rax + 1024], 115 // 's'
    mov     byte ptr [rax + 1026], 116 // 't'
    mov     byte ptr [rax + 1028], 117 // 'u'
    mov     byte ptr [rax + 1030], 118 // 'v'

Each of these correspond to one or more writes to the buffer, as best the compiler could arrange it for performance.

> **Example with 1 byte**
> 
>     buffer[0] = 'a';
>     |
>     V
>     mov     byte ptr [rax], 97

You read that as:

| `mov` | `byte`             | `ptr [rax],`                         | `97`                                                   |
| ----- | ------------------ | ------------------------------------ | ------------------------------------------------------ |
| copy  | into 1 memory cell | found at the address stored in `rax` | the integer with the same value as ascii character `a` |

The syntax might be new to you, but there should be nothing really *surprising* here - move a byte into memory, sure, that's the same as we wrote in the `C++` code that was compiled.

> **Example with 2 bytes**
> 
>     buffer[64] = 'b';
>     buffer[65] = 'c';
>     |
>     V
>     mov     word ptr [rax + 64], 25442

Here, the CPU can take advantage of the fact that the two bytes are next to each other, so it becomes:

| `mov` | `word`                              | `ptr [rax + 64],`                                                 | `25442`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----- | ----------------------------------- | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| copy  | into a 2-long range of memory cells | whose beginning is found 64 down from the address stored in `rax` | an integer with the same value as two adjacent ascii characters 'b'+'c'<br><br>You see here that - exactly because the chars are *right next to each other* and not 4 apart, the CPU can copy 2 chars with 1 instruction.<br><br>There's a bit of math here.<br><br>Much like `24 = 2*10 + 4`, it is also the case that `bc = b*256 + c` (or, on some machines, reversed: `bc = b + c*256`)<br><br>So the compiler does that piece of math and says "instead of first copying `b` into memory and then copying `c` just after, just do 1 copy of `bc` into memory directly. |

This doesn't always work! So next:

> **Example with 3 bytes**
>  
>     buffer[128] = 'd';
>     buffer[129] = 'e';
>     buffer[130] = 'f';
>     |
>     V
>     mov     word ptr [rax + 128], 25956
>     mov     byte ptr [rax + 130], 102

This is two instructions - there's no "copy 3 bytes' instruction, so to copy 3 bytes, it copies 2 + 1. As in the previous example, here `25956` is `de`, and `102` is just the value for ascii character `f`

What if we have more than 3?

> **Example with 4 bytes**
> 
>     buffer[256] = 'g';
>     buffer[257] = 'h';
>     buffer[258] = 'i';
>     buffer[259] = 'j';
>     |
>     V
>     mov     dword ptr [rax + 256], 1785292903

Here, `ghij` becomes `1785292903` - `g*256^3 + h*256^2 + i*256 + j` (or, again, the other way around).

Four characters, four bytes - and luckily the CPU has a `mov dword` command, where `dword` is "2 words," or "2*2 bytes," so we can do it in 1 instruction.

>  **Example with 8 bytes**
> 
>     buffer[512] = 'k';
>     buffer[513] = 'l';
>     buffer[514] = 'm';
>     buffer[515] = 'n';
>     buffer[516] = 'o';
>     buffer[517] = 'p';
>     buffer[518] = 'q';
>     buffer[519] = 'r';
>     |
>     V
>     movabs  rcx, 8246496016588434539
>     mov     qword ptr [rax + 512], rcx

Once again, the compiler has calcuated which 8-byte integer means the same as `klmnopqr` because we do have a command  `mov qword` (quad-word, 4 words, 4*2 bytes)

For some reason, it doesn't just do `mov qword ptr [rax + 512], 8246496016588434539`, I think it has something to do with signed/unsigned integer values but I'm not sure what's actually going on here.

> **Example with 4 bytes with 1-byte padding in between**
> 
>     buffer[1024] = 's';
>     // skip 1025
>     buffer[1026] = 't';
>     // skip 1027
>     buffer[1028] = 'u';
>     // skip 1029
>     buffer[1030] = 'v';
>     // skip 1031
>     |
>     V
>     mov     byte ptr [rax + 1024], 115
>     mov     byte ptr [rax + 1026], 116
>     mov     byte ptr [rax + 1028], 117
>     mov     byte ptr [rax + 1030], 118

...oh this is unfortunate. We're copying 4 bytes in a very predictable pattern, but the best it can do is just to copy one byte. Four times.

I think this might actually be a mistake in the compiler - we didn't *ask* to touch the bytes in between, but they haven't been initialized with any value so the compiler is allowed to put whatever it wants into those spaces in between, and it ought to have done so - I *think*. But I also notice that none of the Big Three compilers do that, so maybe I'm wrong.
