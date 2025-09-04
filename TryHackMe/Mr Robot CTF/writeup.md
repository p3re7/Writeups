# Mr Robot CTF

**Platform:** TryHackMe  
**Difficulty:** Medium  
**IP:** 10.10.32.21 

---

## 1. Información Inicial
- **Objetivo:** Obtener las tres llaves mediante pruebas de penetración.  
- **Herramientas iniciales:** `nmap`, `gobuster`, `wget`, `hydra`, `reverse shell`, `gtfobins`, `crackstation`, `netcat`, `find`.

---

## 2. Reconocimiento y enumeración

Primero realizamos un escaneo básico con `nmap` para identificar los servicios abiertos:
```bash 
nmap -sC -sV -T5 10.10.32.21
```

![Escaneo con nmap para descubrimiento de puertos y servicios](screenshots/1.nmap.png)

Podemos ver como están abiertos los puertos 22, 80 y 443. Vamos a acceder al sitio web por el puerto 80 para ver que encontramos.

![Vista previa del sitio web](screenshots/2.vista_web_principal.png)

En el sitio web no encontramos nada relevante, por lo que realizamos un descubrimiento de directorios con `gobuster`, usando el parámetro `-x` para que nos muestre también ficheros TXT, PHP y HTML.

```bash 
gobuster dir -u http://10.10.32.21 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
```
![Vista previa del sitio web](screenshots/3.gobuster.png)

En el resultado de esta herramienta podemos observar sitios interesantes como:
- /wp-login (inicio de sesión de WordPress)
- /admin
- /license
- /robots.txt

## 3. Explotación

Primero vamos a entrar al fichero `robots.txt` (http://10.10.32.21/robots.txt), donde se hayan dos ficheros. El primero es un diccionario (fsocity.dic) y el segundo es la primera llave de la máquina (key-1-of-3.txt)

![Contenido del fichero robots.txt](screenshots/4.robots.txt.png)

Entramos a ver la primera llave (http://10.10.32.21/key-1-of-3.txt)

![Fichero público del sitio web que contiene la primera llave](screenshots/5.key1.png)

Con el diccionario que nos facilitan, podemos intentar hacer fuerza bruta ingresando el mismo diccionario en usuario y contraseña en el inicio de sesión de WordPress.

![Diccionario fsocity.dic que hemos encontrado en robots.txt](screenshots/6.diccionario.png)

Utilizaremos `Hydra` para realizar el ataque de fuerza bruta a los campos de usuario y contraseña.

Este ataque comienza a ejecutarse pero tardaba demasiado (aunque hubiera funcionado de haber tardado menos).

```bash 
hydra -L fsocity.dic -P fsocity.dic 10.10.32.21 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=Invalid username"
```
![Ataque de fuerza bruta con Hydra a login de WordPress](screenshots/7.intento_hydra_fallido.png)

Como alternativa, seguimos mirando en los resultados de `gobuster` y probando diferentes directorios para ver su contenido.

Encontramos dentro del directorio /license la redirección al fichero license.txt, cuyo contenido se basa en una frase provocadora pero que indica que hay algo más en este fichero.

Accedemos al código fuente y vemos un hash al final de la frase, que por su signo final parece ser `base64`.

![Contenido de fichero en /license](screenshots/8.license_hash.png)

Para decodificar el hash en base64 podemos usar la herramienta `base64` que viene instalada en Kali Linux.

```bash 
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 --decode
```

![Contenido de fichero en /license](screenshots/9.base64_decode.png)

Su resultado es unas credenciales, probablemente para el inicio de sesión del panel de WordPress.

![Inicio de sesión de WordPress](screenshots/10.login-wp.png)

Dando una vuelta por el panel de control nos damos cuenta de que dentro de Appearance podemos editar los Temas, y entre todos los ficheros a modificar en la columna de la derecha tenemos ficheros en PHP.

![Edición de temas de WordPress](screenshots/11.edit_themes.png)

Probando a modificar y guardar el código, no se aprecia que haga ninguna comprobación. Esto quiere decir que podemos ingresar una `reverse shell` y ejecutar el fichero desde su ubicación mientras escuchamos con `netcat`.

![Reverse shell de pentestmonkey](screenshots/12.reverse_pentestmonkeys.png)

La mítica reverse shell de pentestmonkey (https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) nunca falla, por lo que copiaremos el Raw desde GitHub y lo pegaremos en el editor de temas modificando la dirección IP y el puerto que queramos utilizar.

![Reverse shell en editor de temas](screenshots/13themes_with_reverse.png)

Pondremos netcat a la escucha desde nuestra máquina por el puerto 1234.
```bash 
nc -lvnp 1234
```

Desde la dirección http://10.10.32.21/wp-includes/themes/TwentyFifteen/404.php ejecutamos la reverse shell y recibiremos la conexión automáticamente en nuestro netcat.

![Ejecución de reverse shell](screenshots/14.ejecutar_reverse_shell.png)
![Conexión de víctima a netcat](screenshots/15.conexion_nc.png)

## 4. Postexplotación

Ya tenemos Ya tenemos una reverse shell con la máqunia víctima y entramos como el usuario `daemon`. Este usuario puede ver el directorio personal de robot en el que hay dos ficheros:

- key-2-of-3.txt
- password.raw-md5

El primer fichero contiene la segunda llave pero no tenemos permiso para acceder a él y el segundo cotiene las credenciales del usuario robot, pero con la contraseña codificada en MD5.

![Listado de ficheros en directorio de robot](screenshots/16.escala1.png)

El sitio web más conocido para tratar de decodificar un hash MD5 es Crackstation (https://crackstation.net/).

![Decodificando hash md5 con crackstation](screenshots/17.escala2.png)

Conseguimos la contraseña del usuario con la que accederemos a la cuenta de robot y podremos leer el fichero que contiene la segunda llave.

![Decodificando hash md5 con crackstation](screenshots/18.key2.png)

Trataremos de buscar ficheros con permisos SUID que se ejecuten con los permisos del propietario y no del usuario que los ejecuta.

```bash 
find / -perm -u=s -type f 2>/dev/null
```

![Búsqueda de ficheros con permisos SUID](screenshots/19.escala3.png)

Entre todos los ficheros hay uno que no es tan común encontrar, se trata de `/usr/local/bin/nmap`.

Visitaremos la página más común para escalada de privilegios, GTFObins (https://gtfobins.github.io).

En el buscador escribiremos "nmap" y en la sección de Shell leeremos y seleccionaremos la opción más apropiada, que en este caso es la (b) que puede ejecutarse con comandos de shell.

![Consulta de gtfobins para escalada de privilegios](screenshots/20.escala4.png)

Ejecutando esos dos sencillos comandos, podemos ver como la respuesta al comando `whoami` es root, y tenemos privilegios de administrador para acceder a la última llave.

```bash 
nmap --interactive
!sh
```

![Ejecución de escala de privilegios con nmap](screenshots/21.key4.png)

## 5. Conclusiones

- La enumeración exhaustiva con Nmap y Gobuster permitió identificar servicios y directorios críticos sin necesidad de fuerza bruta.
- La máquina mostró la importancia de revisar archivos expuestos como robots.txt.
- Encontrar binarios SUID y revisar permisos de sudo puede llevar a escalada de privilegios rápida.


















