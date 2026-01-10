# Transformation

Author: madStacks

Description:

I wonder what this really is... [enc](https://mercury.picoctf.net/static/77a2b202236aa741e988581e78d277a6/enc) 

`''.join([chr((ord(flag[i]) << 8) + ord(flag[i + 1])) for i in range(0, len(flag), 2)])`

## Solve
flag: `picoCTF{16_bits_inst34d_of_8_75d4898b}`
