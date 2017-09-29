---
layout: post
title:  "EKOPARTY CTF - EKOVM - Resolviendo el reto"
date:   2017-09-29 12:00:00 -0500
categories: jekyll update
---
El reto empieza con la frase _Some secrets will loop forever._ Despues nos deja
descargar el fichero _vm-ekovm.zip_, dentro de el se encuentra _ekovm.raw_.
Dentro no se ve el flag por ningún lado, no esperabamos menos.
Parece un archivo binario normal cuando lo abrimos con un editor hexadecimal,
pero al final de este se encuentra una dirección web:

```hex
00000f90: 0000 0102 6a11 6101 0210 0010 0000 0000  ....j.a.........
00000fa0: 6874 7470 733a 2f2f 6769 7468 7562 2e63  https://github.c
00000fb0: 6f6d 2f73 6b78 2f73 696d 706c 652e 766d  om/skx/simple.vm
```

[Simple.vm][simplevm]

Dentro de ese sitio vemos que se trata de un proyecto de máquina virtual simple.
Tiene su lenguaje ensamblador, compilador, decompilador y el core de la máquina
virtual.

Leyendo un poco la documentación nos damos cuenta que el archivo original que
descargamos de ekoparty ctf es una imagen raw de esta máquina virtual.
Corriendo la imagen orignal solo se queda la máquina pensando:

```
 ctf $ ./simple-vm ekovm.raw
```

Haciendo uso del mismo decompilador que viene incluido en el repositorio
obtenemos esl siguiente archivo.

```
 ctf $ ./decompiler ekovm.raw > ekovm.in
```
```
	store #1, 0x0010
	store #2, 0x1000
	poke #1, #2
	store #1, 0x0000
	store #2, 0x1001
	poke #1, #2

... Aprox 1100 lineas más ...

	store #1, 0x0000
	store #2, 0x116A
	poke #1, #2
	jmp 0x1000
	exit
	exit
	exit
	exit
	DATA 104
	DATA 116
        ...
```

En el podemos ver las instrucciones que son convertidas a opcode mediante el
compilador, asi que podemos hacer modificaciones a nuestro programa. Probamos
quitando el _jmp 0x1000_ y las instrucciones _exit_
```
./compiler ekovm.in && ./simple-vm ekovm.raw
0F99 - op_unknown(68)
0F9A - op_unknown(74)
0F9B - op_unknown(74)
ERROR running script - Register out of bounds
```

Vemos que el programa falla diciendo que no conoce los op codes _68_ y _74_,
los cuales representan _h_ y _t_, el comienzo de _http_, por lo que deducimos
que el programa nunca llega hasta ese punto, así que revertimos los cambios.

Leyendo más en la documentación vemos que la máquina virtual tiene la opción
_DEBUG_ para arrojar más información sobre el proceso.

```
JUMP_TO(Offset:4096 [Hex:1000]
1000 - Parsing OpCode Hex:10
JUMP_TO(Offset:4096 [Hex:1000]
1000 - Parsing OpCode Hex:10
JUMP_TO(Offset:4096 [Hex:1000]
```

Obtenemos una salida interminable con el contenido de arriba, en este punto
vemos que la instrucción _jmp 0x1000_ es la que no nos deja avanzar, y más
interesante aún, nos lleva a una dirección de memoria de la cual no sabemos
nada.

Analizando el código que obtuvimos con el decompilador vemos que el conjunto
de instrucciones es muy parecido a **BASIC**, y leyendo las instrucciones antes
del _jmp_ vemos lo que está pasando.

```
	store #1, 0x0010
	store #2, 0x1000
	poke #1, #2
```

En el registro _#1_ guarda _0x0010_ y en el registro _#2_ guarda _0x1000_ (ese
número ya lo habíamos visto en algún lado). La instrucción _poke_ guarda el
contenido del registro _#1_ en la dirección de memoria a la que apunta el
registro _#2_.

Así que ahí es a donde apunta nuestro _jmp_, a esa dirección de memoría que
contiene el op_code _0x0010_. Ese op code que se encuentra guardado en la
dirección de memoria _0x1000_ corresponde a la instrucción _jmp_. Ahí está
nuestro loop infernal. La dirección de memoria _0x1000_ tiene la instruccion
_jmp 0x1000_, eso nunca nos dejará avanzar.

En este punto podemos ver que lo que hace _ekovm.raw_ es ir escribiendo en
memoria de forma dinámica los op codes, mediante instrucciones store y poke. Y
lo bueno es que lo va haciendo de forma consecutiva, asi que en vez de cargar
un debugger para ver la memoria en cada paso y que es lo que se está ejecutando
podemos copiar los op codes que se guardan en el registro _#1_ con tu editor
favorito, o también puedes imprimirlos en pantalla desde la misma imagen, solo
hace falta agregar `print_int #1` y al final nos entregará una cadena con los
opcodes a ejecutar.

```
10 00 10 01 01 68 00 01 02 2D 00 20 03 01 02 01 04 65 00 01 05 2E 00 20 06 04
05 01 07 6C 00 01 08 23 00 20 09 07 08 01 01 00 00 01 02 00 00 01 03 00 00 01
...
02 00 00 01 03 00 00 01 04 00 00 01 05 00 00 01 06 00 00 01 07 00 00 01 08 00
00 01 09 00 00 01 01 31 00 01 02 5C 00 20 03 01 02 01 04 32 00 01 05 13 00 20
```

Ahora tenemos un archivo imagen con los opcodes que la imagen original
cargaba en memoria, este archivo le podemos hacer el mismo proceso que al
original,

Ahora tenemos un archivo imagen con los opcodes que la imagen original
cargaba en memoria, este archivo le podemos hacer el mismo proceso que al
original, lo decompilamos con la misma herramienta y nos da el siguiente
archivo.

```
	jmp 0x1000
	store #1, 0x0068
	store #2, 0x002D
	xor #3, #1, #2
	store #4, 0x0065
	store #5, 0x002E
	xor #6, #4, #5
	store #7, 0x006C
	store #8, 0x0023
	xor #9, #7, #8
        ...
	store #4, 0x0032
	store #5, 0x0013
	xor #6, #4, #5
```

Ahí está el _jmp_ del infierno, ese simplemente lo removemos, ya no hace falta.
Ya tenemos algo con más cuerpo para trabajar, vemos que ya no hay instrucciones
poke y ahora tenemos instrucciones _XOR_, de está forma evitas que apareciera el
flag en un editor hexadecimal, está cifrado. Haciendo la primeras operaciones
_XOR_ vemos  que _0x0068 ^ 0x002D =  0x0045_ o lo que es igual, una _E_. El
siguiente resultado es una _K_, el siguiente una _O_. De aquí en adelante solo
hace falta realizar la demás operaciones.


[simplevm]: https://github.com/skx/simple.vm
