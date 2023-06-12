## to_rev

```shell
──╼ $file to_rev
to_rev: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=962f2e5080b7bfb815105615e3b32c77b9822b4d, for GNU/Linux 3.2.0, not stripped
┌─[luc@parrot]─[~/Desktop/LNE/to_rev]
└──╼ $./to_rev
./to_rev: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./to_rev)
┌─[✗]─[luc@parrot]─[~/Desktop/LNE/to_rev]
└──╼ $

```

On cherche alors sur internet la libc 2.34 puis le loader asscocié à cette libc puis on patch le binaire avec `pwninit`

```shell
└──╼ $pwninit 
bin: ./to_rev
libc: ./libc.so(6).6
ld: ./ld-linux-x86-64.so.2

symlinking ./libc.so.6 -> libc.so(6).6
copying ./to_rev to ./to_rev_patched
running patchelf on ./to_rev_patched
┌─[luc@parrot]─[~/Desktop/LNE/to_rev]
└──╼ $ll
total 6.7M
drwxr-xr-x 1 luc luc  156 Jun  9 06:25  caesar
-rw-r--r-- 1 luc luc   35 Jun  2 04:48  flag.txt
-rwxrwxrwx 1 luc luc 236K Jun  8 13:10  ld-linux-x86-64.so.2
lrwxrwxrwx 1 luc luc   12 Jun 11 19:00  libc.so.6 -> 'libc.so(6).6'
-rw-rw-rw- 1 luc luc 6.4M Jun  8 13:13 'libc.so(6).6'
drwxr-xr-x 1 luc luc  176 Jun  9 04:29  logical
-rwxr-xr-x 1 luc luc 2.3K Jun  9 11:05  solve.py
-rwxr-xr-x 1 luc luc  17K May 12 11:33  to_rev
-rwxr-xr-x 1 luc luc  25K Jun 11 19:00  to_rev_patched
drwxr-xr-x 1 luc luc    0 Jun 11 18:58 'untitled folder'
┌─[luc@parrot]─[~/Desktop/LNE/to_rev]
└──╼ $./to_rev_patched 

    ███████╗██╗     ███████╗██╗  ██╗██╗███╗   ██╗ ██████╗      █████╗ ██╗
    ██╔════╝██║     ██╔════╝╚██╗██╔╝██║████╗  ██║██╔════╝     ██╔══██╗██║
    █████╗  ██║     █████╗   ╚███╔╝ ██║██╔██╗ ██║██║  ███╗    ███████║██║
    ██╔══╝  ██║     ██╔══╝   ██╔██╗ ██║██║╚██╗██║██║   ██║    ██╔══██║██║
    ██║     ███████╗███████╗██╔╝ ██╗██║██║ ╚████║╚██████╔╝    ██║  ██║██║
    ╚═╝     ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝ ╚═════╝     ╚═╝  ╚═╝╚═╝

Heyo whatup my dude ?! Prove me you're good or get the f*** out !
 > AAAAAAAAAAAAAAAAAA
...I am way too awesome to hangout with you, see ya loser!

```

On a maintenant notre binaire qui fonctionne
Un strings sur le binair enosu donne ceci:
```
Heyo whatup my dude ?! Prove me you're good or get the f*** out !
%36s
1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!
What the fock you so good bruhhhhhh!
Accept some of my drip please :
/bin/cat flag.txt
...I am way too awesome to hangout with you, see ya loser!
```
On croirait que `1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!` est le flag mais en le passant en entrée au binaire, c'est non on obtient ceci `...I am way too awesome to hangout with you, see ya loser!` qui veut dire perdu, donc va falloir creuser plus.
On charge le binaire dans ghidra pour l'analyser:

### Main
```c

undefined8 main(void)

{
  int iVar1;
  undefined8 uVar2;
  char *__s1;
  long in_FS_OFFSET;
  undefined local_38 [40];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  printf("%s\n\n",&DAT_00102008);
  printf("Heyo whatup my dude ?! Prove me you\'re good or get the f*** out !\n > ");
  __isoc99_scanf(&DAT_0010248e,local_38);
  uVar2 = replace(local_38);
  uVar2 = caesar(uVar2);
  __s1 = (char *)logical(uVar2);
  iVar1 = strcmp(__s1,"1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!");
  if (iVar1 == 0) {
    puts("\nWhat the fock you so good bruhhhhhh!\nAccept some of my drip please :\n");
    system("/bin/cat flag.txt");
  }
  else {
    puts("...I am way too awesome to hangout with you, see ya loser!");
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```
On ppeut voir notre entrée passe par trois fonctions, `replace, caesar, et logical` , la sortie final est ensuiet comparé avec la chaine `1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!`, 
si les deux chaines sont égaux, on affiche ceci `\nWhat the fock you so good bruhhhhhh!\nAccept some of my drip please :\n` puis le flag, sinon on affiche le message `...I am way too awesome to hangout with you, see ya loser!`

On va maintenant analyser chacunes de ces fonctions. J'ai décider de commencer par la fonction la plus basse `logical` cela me permettra de comparer directement la sortie de mon entrée avec `1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!`
J'ai donc décider donc de patcher le binaire afin de n'avoir que le `logical` comme fonction à passer et ensuite je commence mon analyse:

##### *Logical*
```c
byte * logical(char *param_1)

{
  byte bVar1;
  byte *__dest;
  size_t sVar2;
  long in_FS_OFFSET;
  int local_5c;
  byte *local_58;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined4 local_28;
  long local_20;
  
  local_20 = *(long *)(in_FS_OFFSET + 0x28);
  if (param_1 == (char *)0x0) {
    __dest = (byte *)0x0;
  }
  else {
    sVar2 = strlen(param_1);
    __dest = (byte *)malloc(sVar2);
    local_48 = 0x2a6c56295f091744;
    local_40 = 0x316f75022d464243;
    local_38 = 0x6e147b433b6e5331;
    local_30 = 0x7e5f541d3b6b025d;
    local_28 = 0x4b114b;
    if (__dest != (byte *)0x0) {
      strcpy((char *)__dest,param_1);
      local_58 = __dest;
      for (local_5c = 0; sVar2 = strlen(param_1), (ulong)(long)local_5c < sVar2;
          local_5c = local_5c + 1) {
        bVar1 = *local_58;
        sVar2 = strlen((char *)&local_48);
        *local_58 = bVar1 ^ *(byte *)((long)&local_48 + (ulong)(long)local_5c % sVar2);
        local_58 = local_58 + 1;
      }
    }
  }
  if (local_20 == *(long *)(in_FS_OFFSET + 0x28)) {
    return __dest;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```
~~On peut voir ici que la fonction prend en entrée l'addresse de notre chaine et verifie ensuite si cette addresse n'est pas *NULL*, si elle est *NULL* alors on mets `__dest` (pour destination) à `NULL` puis on return `__dest` , sinon ... ~~
A cet instant j'ai décider d'utiliser le fameux chatGPT pour aller plus vite:
```
The given function appears to take a parameter `param_1`, which is a pointer to a character array (`char*`). It performs some operations on the input string and returns a pointer to a dynamically allocated byte array (`byte*`).

1.  It checks if `param_1` is a null pointer and returns a null pointer in that case.
2.  It calculates the length of the input string using `strlen`.
3.  It allocates memory for a byte array using `malloc` with a size equal to the length of `param_1`.
4.  It copies the contents of `param_1` into the newly allocated byte array using `strcpy`.
5.  It performs a bitwise XOR operation on each byte of the byte array using a predefined byte sequence.
6.  It returns the resulting byte array.
```
Génial !! Mais pas vraiment de précision sur avec quoi il fait le bitwise XOR, en regardnant nous même, on s'aperçoit que l'opération XOR est effectué entre chaque caractère de notre input et `local48[i]` , donc si par exemple on est à la 10ème itération, on aura `input[10] ^ local[10]` et `local48[10]` tombe dans `local40` , j'ai compris aussitôt alors que le XOR s'effectue avec tout l'array. Il suffira donc de faire un XOR entre `1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!` et cet long array `\x4b\x11\x4b\x7e\x5f\x54\x1d\x3b\x6b\x02\x5d\x6e\x14\x7b\x43\x3b\x6e\x53\x31\x31\x6f\x75\x02\x2d\x46\x42\x43\x2a\x6c\x56\x29\x5f\x09\x17\x44` pour trouver l'input initial. Voici une petite fonction qui nous le fait:
```python
def logical(output):
	byte_sequence = bytearray(b'\x4b\x11\x4b\x7e\x5f\x54\x1d\x3b\x6b\x02\x5d\x6e\x14\x7b\x43\x3b\x6e\x53\x31\x31\x6f\x75\x02\x2d\x46\x42\x43\x2a\x6c\x56\x29\x5f\x09\x17\x44') # Byte sequence used
	byte_sequence.reverse()
	input_bytes = bytearray(len(output))
	for i in range(len(output)):
		input_bytes[i] = output[i] ^ byte_sequence[i % len(byte_sequence)]
	return input_bytes
```
En l'exécutant:
```sh
──╼ $python3 solve.py 
bytearray(b'uHeo_e3xp4u_q00t_4_UpHf_3e4_H0l_j0j')
```
puis dans le binaire patché: 
```
└──╼ $./to_rev_logical_patched 

    ███████╗██╗     ███████╗██╗  ██╗██╗███╗   ██╗ ██████╗      █████╗ ██╗
    ██╔════╝██║     ██╔════╝╚██╗██╔╝██║████╗  ██║██╔════╝     ██╔══██╗██║
    █████╗  ██║     █████╗   ╚███╔╝ ██║██╔██╗ ██║██║  ███╗    ███████║██║
    ██╔══╝  ██║     ██╔══╝   ██╔██╗ ██║██║╚██╗██║██║   ██║    ██╔══██║██║
    ██║     ███████╗███████╗██╔╝ ██╗██║██║ ╚████║╚██████╔╝    ██║  ██║██║
    ╚═╝     ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝ ╚═════╝     ╚═╝  ╚═╝╚═╝

Heyo whatup my dude ?! Prove me you're good or get the f*** out !
 > uHeo_e3xp4u_q00t_4_UpHf_3e4_H0l_j0j

What the fock you so good bruhhhhhh!
Accept some of my drip please :

/bin/cat: flag.txt: No such file or directory
┌─[luc@parrot]─[~/Desktop/LNE/to_rev/logical]
└──╼ $
```

##### *Caesar*
Ghidra
```c
char * caesar(char *param_1)

{
  char *__dest;
  size_t __size;
  char *local_18;
  
  if (param_1 == (char *)0x0) {
    __dest = (char *)0x0;
  }
  else {
    __size = strlen(param_1);
    __dest = (char *)malloc(__size);
    if (__dest != (char *)0x0) {
      strcpy(__dest,param_1);
      for (local_18 = __dest; *local_18 != '\0'; local_18 = local_18 + 1) {
        if ((('`' < *local_18) && (*local_18 < '{')) || (('@' < *local_18 && (*local_18 < '[')))) {
          if ((*local_18 < 'n') && ((*local_18 < 'N' || ('Z' < *local_18)))) {
            *local_18 = *local_18 + '\r';
          }
          else {
            *local_18 = *local_18 + -0xd;
          }
        }
      }
    }
  }
  return __dest;
}
```

Pareil , on demande à ChatGPT:
```
The `caesar` function takes a parameter `param_1`, which is a pointer to a character array (`char*`). It performs the following operations:

1.  It checks if `param_1` is a null pointer and returns a null pointer in that case.
2.  It calculates the length of the input string using `strlen`.
3.  It allocates memory for a character array using `malloc` with a size equal to the length of `param_1`.
4.  It copies the contents of `param_1` into the newly allocated character array using `strcpy`.
5.  It iterates through each character of the character array and performs a Caesar cipher decryption operation.
 -   It checks if the character is within the range ('`' to `'{'`) or (`'@'` to `'['`).
    -   If the character is within this range and less than `'n'`, and is less than  'N' or greater than `'Z'`), it subtracts `'\r'` (13) from the character.
    -   Otherwise, it subtracts `0xd` (13) from the character.
6.  It returns the resulting character array.
```

Voici la fonction que j'ai écrite pour reverse:
```python
def ceasar(output):
	input_bytes = bytearray(len(output))
	minus = list(map(lambda i: i - ord('\r'), output))
	plus = list(map(lambda i: i +13, output))
	final=[]
	for i in range(len(minus)):
		if ( (ord('`') < minus[i] < ord('{')) or (ord('@') < minus[i] < ord('['))):
			if( minus[i] < ord('n') and (minus[i]< ord('N') or ord('Z')< minus[i])):
				## Minus fit the two conditions great it's definitely the minus one
				final.append(minus[i])
			else:
				if ( (ord('`') < output[i] < ord('{')) or (ord('@') < output[i] < ord('[')) ):
					final.append(plus[i])
				else:
					final.append(output[i])
		elif( ( ord('`') < plus[i] < ord('{')) or (ord('@') < plus[i] < ord('[') )):
			if ( (ord('`') < output[i] < ord('{')) or (ord('@') < output[i] < ord('[')) ):
				final.append(plus[i])
			else:
				final.append(output[i])
		else:
			final.append(output[i])
	return bytearray(final)
```

###### Explication: 
J'ai d'abord créé 2 listes qui contiennent le décalage +13 et -13 pour notre bytearray, l'objectif étant de verifier les bytes qui respectent les conditions définsi et de choisir le bon en conséquence.
On itère ensuite sur notre input. 
	1. On verifie si le byte représentant notre input décalé de -13 est compris entre '`' et  `'{' ou entre '@' et `'['`
		1. Si Oui ?? On verifie ensuite si ce byte est plus petit que 'n' and (plus petit que 'N' ou plus grand que 'Z') 
			1. Oui ? Alors c'est le byte qu'on cherche
			2. Non ? alors on verifie si notre input lui même n'est pas dans ce range là ( compris entre '`' et  `'{' ou entre '@' et `'['` )
				1. Oui ? Puisque il n'est pas dans le deuxième range (plus petit que 'n' and (plus petit que 'N' ou plus grand que 'Z'), alors c'est forcément notre input décalé de +13 qui marchera dans ce cas , puisque une fois passé il renverra tout simplement notre input
				2. Non ? Notre input n'a pas subit de modificaion donc ( c'est son décalé de -13 qui tombe dans ce range)
	2. Si Non  Si? On verifie si le byte représentant notre input décalé de +13 est compris entre '`' et  `'{' ou entre '@' et `'['`
			1. oui ? Pour eviter les faux positifs, on verifie si notre input lui même est dans ce range
				1. si oui, nul doute l'input recherhé est notre input décalé de +13, 
				2. Non ? Alors faux positif, notre input n'a pas reçu de modif (c'est son décalé de +13 qui tombe dans ce range)
	3. Sinon notre input n'a pas eu de modification

Après exécution, 
```shell
─╼ $python3 solve.py 
bytearray(b'hUrb_r3kc4h_d00g_4_HcUs_3r4_U0y_w0w')
```
Puis sur le binaire patché , 
```
──╼ $./to_rev_caesar_patched 

    ███████╗██╗     ███████╗██╗  ██╗██╗███╗   ██╗ ██████╗      █████╗ ██╗
    ██╔════╝██║     ██╔════╝╚██╗██╔╝██║████╗  ██║██╔════╝     ██╔══██╗██║
    █████╗  ██║     █████╗   ╚███╔╝ ██║██╔██╗ ██║██║  ███╗    ███████║██║
    ██╔══╝  ██║     ██╔══╝   ██╔██╗ ██║██║╚██╗██║██║   ██║    ██╔══██║██║
    ██║     ███████╗███████╗██╔╝ ██╗██║██║ ╚████║╚██████╔╝    ██║  ██║██║
    ╚═╝     ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝ ╚═════╝     ╚═╝  ╚═╝╚═╝

Heyo whatup my dude ?! Prove me you're good or get the f*** out !
 > hUrb_r3kc4h_d00g_4_HcUs_3r4_U0y_w0w

What the fock you so good bruhhhhhh!
Accept some of my drip please :

/bin/cat: flag.txt: No such file or directory

```

##### *replace*

```c

char * replace(char *param_1)

{
  char *__dest;
  size_t sVar1;
  int local_24;
  char *local_18;
  
  if (param_1 == (char *)0x0) {
    __dest = (char *)0x0;
  }
  else {
    sVar1 = strlen(param_1);
    __dest = (char *)malloc(sVar1);
    if (__dest != (char *)0x0) {
      strcpy(__dest,param_1);
      local_24 = 0;
      sVar1 = strlen(param_1);
      for (local_18 = __dest; *local_18 != '\0'; local_18 = local_18 + 1) {
        *local_18 = param_1[(long)((int)sVar1 - local_24) + -1];
        local_24 = local_24 + 1;
      }
    }
  }
  return __dest;
}
```

Encore chatGPT
```
The `replace` function takes a parameter `param_1`, which is a pointer to a character array (`char*`). It performs the following operations:

1.  It checks if `param_1` is a null pointer and returns a null pointer in that case.
2.  It calculates the length of the input string using `strlen`.
3.  It allocates memory for a character array using `malloc` with a size equal to the length of `param_1`.
4.  It copies the contents of `param_1` into the newly allocated character array using `strcpy`.
5.  It iterates through each character of the character array and replaces each character with the character from `param_1` in reverse order.
6.  It returns the resulting character array.
```

Rien de plus simple, il suffit de renverser la chaine de départ : 
```python
def replace(output):
	return output[::-1]
```

Et bingo :

```shell
──╼ $python3 solve.py 
bytearray(b'w0w_y0U_4r3_sUcH_4_g00d_h4ck3r_brUh')
```

Et si on exécute le binaire : 
```shell
[✗]─[luc@parrot]─[~/Desktop/LNE/to_rev]
└──╼ $./to_rev_patched 

    ███████╗██╗     ███████╗██╗  ██╗██╗███╗   ██╗ ██████╗      █████╗ ██╗
    ██╔════╝██║     ██╔════╝╚██╗██╔╝██║████╗  ██║██╔════╝     ██╔══██╗██║
    █████╗  ██║     █████╗   ╚███╔╝ ██║██╔██╗ ██║██║  ███╗    ███████║██║
    ██╔══╝  ██║     ██╔══╝   ██╔██╗ ██║██║╚██╗██║██║   ██║    ██╔══██║██║
    ██║     ███████╗███████╗██╔╝ ██╗██║██║ ╚████║╚██████╔╝    ██║  ██║██║
    ╚═╝     ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚═╝  ╚═══╝ ╚═════╝     ╚═╝  ╚═╝╚═╝

Heyo whatup my dude ?! Prove me you're good or get the f*** out !
 > w0w_y0U_4r3_sUcH_4_g00d_h4ck3r_brUh

What the fock you so good bruhhhhhh!
Accept some of my drip please :

CTF{fake_flag_fake_flag_fake_flag}

```




## Code

```python
def logical(output):
#output = b'1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!' # Desired output
	byte_sequence = bytearray(b'\x4b\x11\x4b\x7e\x5f\x54\x1d\x3b\x6b\x02\x5d\x6e\x14\x7b\x43\x3b\x6e\x53\x31\x31\x6f\x75\x02\x2d\x46\x42\x43\x2a\x6c\x56\x29\x5f\x09\x17\x44') # Byte sequence used
	byte_sequence.reverse()
	input_bytes = bytearray(len(output))
	for i in range(len(output)):
		input_bytes[i] = output[i] ^ byte_sequence[i % len(byte_sequence)]
	return input_bytes

  
def ceasar(output):
	input_bytes = bytearray(len(output))
	minus = list(map(lambda i: i - ord('\r'), output))
	plus = list(map(lambda i: i +13, output))
	final=[]
	for i in range(len(minus)):
		if ( (ord('`') < minus[i] < ord('{')) or (ord('@') < minus[i] < ord('['))):
			if( minus[i] < ord('n') and (minus[i]< ord('N') or ord('Z')< minus[i])):
				## Minus fit the two conditions great it's definitely the minus one
				final.append(minus[i])
			else:
				if ( (ord('`') < output[i] < ord('{')) or (ord('@') < output[i] < ord('[')) ):
					final.append(plus[i])
				else:
					final.append(output[i])
		elif( ( ord('`') < plus[i] < ord('{')) or (ord('@') < plus[i] < ord('[') )):
			if ( (ord('`') < output[i] < ord('{')) or (ord('@') < output[i] < ord('[')) ):
				final.append(plus[i])
			else:
				final.append(output[i])
		else:
			final.append(output[i])
	return bytearray(final)


def replace(output):
	return output[::-1]

output = b'1_l0v3_R3v3rsE_Eng1n33r1ng_dUd3!!!!' # Desired output
print(logical(output))
print( ceasar(logical(output)))
print( replace(ceasar(logical(output)) ) )
```
