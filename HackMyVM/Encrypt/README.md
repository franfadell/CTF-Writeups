# Write-Up: Máquina Encrypt 

Configuramos la máquina “Encrypt” con el adaptador 1 y 2 en adaptador de solo anfitrión. A Kali Linux lo configuramos con adaptador 1 en NAT y el 2 en solo anfitrión. 
Primero ejecutamos “Encrypt” y luego Kali Linux.

## 1. Reconocimiento y Escaneo de Red

En Kali Linux:
Ejecutamos: 
```bash
sudo arp-scan -I eth1 --localnet
```
sudo ejecuta comandos como administrador, arp-scan es una herramienta que envía paquetes ARP para descubrir todos los dispositivos conectados a la red local (los paquetes ARP se usan para asociar direcciones IP con direcciones MAC), -I eth1 especifica la interfaz de red que vamos a usar (usamos eth1 en lugar de eth0 u otro ya que en VirtualBox configuramos “Encrypt” con conexión de solo anfitrión que es una red privada y aislada y es el único canal donde nuestras maquinas se conectan), --localnet indica a la herramienta que calcule y escanee toda la subred a la que está conectada esa interfaz.

<img width="730" height="270" alt="image7" src="https://github.com/user-attachments/assets/d3d6f6d5-9fd9-4c75-af4a-a986cc3a8b3f" />

Con todas las IPs vamos a descartar las que nos son de “Encrypted”: En la primera linea **IPv4: 192.168.63.6** → esta es la IP de nuestra propia maquina.
Las que terminan en .1 y .2 corresponden al repartidor de IPs de VirtualBox.
**192.168.63.4** y **192.168.5** son las IPs sospechosas (DUP: 2 es VirtualBox recibiendo la misma respuesta ARP dos veces, no es un error).

Para saber realmente cual de esas opciones es, ejecutamos: 
```bash
sudo nmap -F 192.168.63.4 192.168.63.5 
```
Este comando -F (Fast) le pidió a Nmap que revisará los 100 puertos más comunes en lugar de los 65.535 totales, esto es para obtener una respuesta rápida. La IP que muestre puertos abiertos es nuestro objetivo.

<img width="527" height="353" alt="image14" src="https://github.com/user-attachments/assets/2fdc6090-cafb-4d87-a51e-fcf6f17f0f2e" />

Podemos observar que la dirección MAC es la misma para ambas IPs (eso explica el DUP: 2). Esto significa que la máquina Encrypt está respondiendo a 2 IPs, elegimos **192.168.63.4**.
El puerto 22 (SSH) está abierto, es el servicio Secure Shell usado para administrar sistemas operativos de forma remota.
El puerto 443 (HTTPS) tiene un servidor web cifrado corriendo.

Ahora queremos saber qué es lo que hay detrás de los puertos abiertos, ejecutamos: 
```bash
sudo nmap -p 22,443 -sC -sV 192.168.63.4 
```
-sC extrae las versiones exactas del software y -sV ejecutará scripts de Nmap, al ser un HTTPS intentará extraer información útil.

<img width="767" height="372" alt="image3" src="https://github.com/user-attachments/assets/2f071bb4-52e5-4070-bb1c-2ff9f290bed3" />

## 2. Explotación Inicial

Miremos esta línea, `ssl-cert: Subject: commonName=iot:Goat123!` 
Cuando un administrador configura un sitio web por HTTPS debe generar un certificado SSL, el campo commonName generalmente se usa para poner el nombre del dominio. Pero aquí el creador de la máquina cometió un error y dejó escrito **iot:Goat123!**. Esa sintaxis (user:password) es el formato estándar para presentar credenciales. Tenemos un posible usuario que probaremos para una explotación inicial.

Ejecutamos: 
```bash
ssh iot@192.168.63.4
```

<img width="776" height="287" alt="image10" src="https://github.com/user-attachments/assets/042ec4e7-b25b-4720-9c0c-30e2e335dbb5" />

Al ejecutar el comando, nos da un mensaje de advertencia porque nos conectamos a una máquina nueva. Simplemente escribimos “yes” y luego escribimos la contraseña que encontramos antes “Goat123!”. Ahora estamos dentro del sistema operativo de la máquina víctima y podemos comenzar la fase de Post-Exploitation y Escalada de privilegios.

Ejecutamos: 
```bash
ls -la
```
Para ver los archivos

<img width="461" height="160" alt="image5" src="https://github.com/user-attachments/assets/6fb39ae3-5ddc-4da4-b935-0c86f980d550" />

Leemos `user.txt` con el comando cat: 
```bash
cat user.txt
```

<img width="522" height="30" alt="image1" src="https://github.com/user-attachments/assets/07c205ad-1198-46cd-93ed-be88443dbeac" />

`user.txt` contiene lo siguiente:
`8a86e2c71318c41c30c442a0605af4e638de10f2476a70d0d054c19070db1a9d`

## 3. Escalada de Privilegios

Ahora verificamos si el administrador real cometió el error de permitirnos ejecutar comandos con privilegios elevados. Ejecutamos:
```bash
sudo -l
```

<img width="706" height="65" alt="image8" src="https://github.com/user-attachments/assets/06fcdf5a-f40a-46b4-b223-45a2935f4f59" />

Nos pide la contraseña que es Goat123! pero sin embargo no tenemos permiso para usar sudo.

A veces los usuarios dejan contraseñas o comandos sensibles en el historial de su consola, la revisamos con:
```bash
cat .bash_history
```

<img width="310" height="50" alt="image11" src="https://github.com/user-attachments/assets/4e9fd944-29c9-4811-b09c-5c3008f0361d" />

El historial está vacío.

Algunos programas necesitan permisos de root para funcionar temporalmente, a veces los administradores le dan esos permisos a programas que no deberían tenerlo, lo que nos permite abusar de esto. Usamos: 
```bash
find / -perm -4000 -type f 2>/dev/null
```
Busca find desde la raíz / todos los archivos -type f que tengan el permiso especial SUID -perm -4000. El 2>/dev/null simplemente oculta los errores para que el resultado sea fácil de leer. 

<img width="445" height="207" alt="image13" src="https://github.com/user-attachments/assets/9fd9d5cd-ffda-4134-8156-e1d663ec0d81" />

Todos los binarios SUID son los que vienen por defecto en cualquier instalación de Debian.

A veces, el usuario administrador deja scripts programados para que se ejecuten automáticamente cada cierto tiempo. Si podemos modificar este script, cuando el sistema lo ejecute, ejecutará lo que nosotros queramos con privilegios máximos. 
Ejecutamos: 
```bash
cat /etc/crontab 
```
Este comando lee el archivo principal donde se configuran las tareas repetitivas del sistema. 

<img width="832" height="372" alt="image15" src="https://github.com/user-attachments/assets/da41ddfe-fc30-452e-a694-9685ad0eb7a8" />

El archivo crontab solo tiene tareas estándar de mantenimiento del sistema.

Pertenecer a ciertos grupos te da acceso a carpetas sensibles o te permite escalar privilegios indirectamente. Ejecutamos:
```bash
id 
```

<img width="512" height="32" alt="image6" src="https://github.com/user-attachments/assets/0d21d703-72ff-49c9-b807-6121a8e3b420" />

iot es un usuario estándar del sistema. Pertenece únicamente a su propio grupo (iot) y al grupo genérico de usuarios (users). 
Sabemos que hay un servidor web corriendo, y a veces los administradores dejan archivos de configuración con credenciales o copias de seguridad de bases de datos en la ruta web pública o en directorios superiores. Ejecutamos: 
```bash
ls -la /var/www/html
```

<img width="452" height="81" alt="image4" src="https://github.com/user-attachments/assets/b695204f-a7a8-4741-987a-c5d8c9e98c67" />

Solo nos sale el index.html generico.
Buscamos archivos relacionados al nombre de la máquina. Le pedimos al sistema que busque cualquier archivo en todo el disco duro que tenga extensiones relacionadas con cifrado o contraseñas. Ejecutamos: 
```bash
find / -type f \( -name "*.enc" -o -name "*.gpg" -o -name "*.kdbx" -o -name "*.zip" \) 2>/dev/null
```

<img width="916" height="35" alt="image12" src="https://github.com/user-attachments/assets/32af6189-dc8b-4e7b-bb50-cb1db55b1833" />

El archivo `text.enc` es simplemente un archivo de fuentes de texto del sistema.

Las Capabilities son una forma de dar permisos de administrador a un programa sin usar el permiso SUID que ya revisamos. A veces, los administradores le dan capacidades de lectura de archivos a programas de copias de seguridad o lenguajes de programación. Ejecutamos: 
```bash
/sbin/getcap -r / 2>/dev/null 
```
Busca recursivamente -r desde la raíz / cualquier archivo binario que tenga capacidades ocultas asignadas. 

<img width="922" height="65" alt="image9" src="https://github.com/user-attachments/assets/c6c4dabe-0be3-410c-8cec-e3e70899eeb8" />

La herramienta getcap nos reveló que el intérprete de Ruby (`/usr/bin/ruby3.3`) tiene asignada la capacidad `cap_setuid=ep`. Esto significa que quien sea que ejecute este programa, puede usarlo para cambiar su Identificador de Usuario (UID) a cualquier otro número del sistema, incluyendo el 0, que corresponde a root.

Como Ruby es un lenguaje de programación muy flexible, podemos enviarle una sola línea de código pidiéndole que nos cambie el ID a 0 e inmediatamente nos abra una consola nueva con esos privilegios máximos.
Ejecutamos: 
```bash
/usr/bin/ruby3.3 -e 'Process::Sys.setuid(0); exec "/bin/bash"' 
```
`-e` Le indica a Ruby que ejecute el texto que le pasamos entre comillas sin necesitar un archivo .rb, `Process::Sys.setuid(0)` aprovecha la capacidad que descubrimos para cambiar el ID al del superusuario, `exec "/bin/bash"` reemplaza el proceso actual por una nueva consola interactiva, la cual heredará los privilegios de root que acabamos de asignarnos. 

<img width="618" height="80" alt="image2" src="https://github.com/user-attachments/assets/7ebfc692-59b9-42af-b5ef-23930fe2ae84" />

El comando funcionó, con `whoami` verificamos que somos root y finalmente leemos `root.txt` y completamos la máquina. 
`bc06e6ee33aee532901fc6cd2926ea0789751a6bfd47f7dea5654cdf68418b85`
