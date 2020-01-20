# Sistemas de ficheros avanzados ZFS

### Creación del escenario

Vamos a crear un escenario que incluya una máquina y varios discos asociados a ella de 500MB.

###### Hemos añadidos los discos `sdb`, `sdc`, `sdd` y `sde`
~~~
lsblk -l
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda    8:0    0    8G  0 disk 
sda1   8:1    0    7G  0 part /
sda2   8:2    0    1K  0 part 
sda5   8:5    0 1022M  0 part [SWAP]
sdb    8:16   0  500M  0 disk 
sdc    8:32   0  500M  0 disk 
sdd    8:48   0  500M  0 disk 
sde    8:64   0  500M  0 disk 
sr0   11:0    1 1024M  0 rom  
~~~

### Instalación de ZFS

Vamos a instlar ZFS, pero no existe ningun paquete en los repositorios de Debian, por lo que no podemos utilizar `apt`.

Tenemos que compilar el paquete. Para compilarlo tenemos que descargar las dependencias y clonamos un repositorio de Github, donde esta alojado dicho paquete.

###### Descargamos las dependecias de ZFS
~~~
apt install build-essential autoconf automake libtool gawk alien fakeroot ksh
apt install zlib1g-dev uuid-dev libattr1-dev libblkid-dev libselinux-dev libudev-dev
apt install libacl1-dev libaio-dev libdevmapper-dev libssl-dev libelf-dev
apt install linux-headers-$(uname -r)
apt install libpython3-dev libpython3.7 libpython3.7-dev linux-headers-amd64 python3-cffi-backend python3-dev python3-distutils python3-lib2to3 python3-ply python3-pycparser python3-setuptools python3.7-dev
~~~

###### Descargamos el repositorio de ZFS
~~~
wget https://github.com/zfsonlinux/zfs/releases/download/zfs-0.8.2/zfs-0.8.2.tar.gz
~~~

###### Descomprimimos y cambiamos los permisos
~~~
tar axf zfs-0.8.2.tar.gz
cd zfs-0.8.2
chown -R root:root ./
chmod -R o-rwx ./
~~~

Ahora vamos a configurar una serie de paramatros, dentro del directorio descargado, para que se realice la compilación y se instale en el directorio indicado y sepa donde buscar las diferentes librerías necesarias.

###### Realizamos la configuración
~~~
./configure \
    --disable-systemd \
    --enable-sysvinit \
    --disable-debug \
    --with-spec=generic \
    --with-linux=$(ls -1dtr /usr/src/linux-headers-*.*.*-common | tail -n 1) \
    --with-linux-obj=$(ls -1dtr /usr/src/linux-headers-*.*.*-amd64 | tail -n 1)
~~~

Si se ha configurado bien, lo siguiente es realizar la compilación y luego de este si todo sale bien, se realizará la instalación del mismo.

###### Empezamos la compilación y la instalación del paquete
~~~
make -j1 && make install
~~~

Para que se inicien los diferentes servicios de zfs tenemos que instalarlo en el directorio `/etc/init.d/`.

###### Instalar script
~~~
cd /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-import /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-mount /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-share /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-zed /etc/init.d/
~~~

El paquete se instala por defecto en el directorio `/usr/local `. Y lo siguiente que vamos a hacer es habilitar los sevicios de ZFS

###### Habilitar los servicios
~~~
update-rc.d zfs-import defaults
update-rc.d zfs-mount defaults
update-rc.d zfs-share defaults
update-rc.d zfs-zed defaults
~~~

El móduo de ZFS no se activa de manera automática, por consiguiente tenemos que activarlo manualmente.

###### Activamos el módulo
~~~
modprobe zfs
~~~

###### Comprobamos que se ha activado correctamente
~~~
lsmod | grep zfs
    zfs                  3760128  0
    zunicode              335872  1 zfs
    zlua                  172032  1 zfs
    zcommon                90112  1 zfs
    znvpair                90112  2 zfs,zcommon
    zavl                   16384  1 zfs
    icp                   311296  1 zfs
    spl                   114688  5 zfs,icp,znvpair,zcommon,zavl
~~~

###### Reiniciamos el equipo
~~~
reboot
~~~

### Gestiona los discos adicionales con ZFS

--------------------------------------------------------------------
* Configurar los discos en RAID, haciendo pruebas de fallo de algún disco y sustitución, restauración del RAID. Comentar ventajas e inconvenientes respecto al uso de RAID software con mdadm.
--------------------------------------------------------------------

#### Creamos un Raid 1

Tenemos que crear un _pool_ de almacenamiento, ya que estos son una colección lógica de dispositivos que proveen espacio para los datasets. Pero el pool que vamos a crear va a ser de tipo `mirror`, que hace referencia a un RAID 1 y en este indicaremos tres de los disco que añadimos anteriormente y el cuarto lo dejaremos de reserva.

###### Creamos un nuevo Pool con Raid 1
~~~
zpool create -f Raid1 mirror /dev/sdb /dev/sdc /dev/sde
~~~

Si queremos cambiar el punto de montaje del Raid, que por defecto se crea en el raíz, podemos hacerlo indicandole la nueva ruta y el nombre del `pool`.

###### Cambiamos el punto de montaje
~~~
zfs set mountpoint=/mnt Raid1
~~~

###### Comprobamos que se ha cambiado el punto de montaje
~~~
zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
Raid1   124K   352M       24K  /mnt
~~~

###### Comprobamos el estado del nuevo Raid
~~~
zpool status
      pool: Raid1
     state: ONLINE
      scan: none requested
    config:

    	NAME        STATE     READ WRITE CKSUM
    	Raid1       ONLINE       0     0     0
    	  mirror-0  ONLINE       0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     ONLINE       0     0     0
    	    sde     ONLINE       0     0     0

    errors: No known data errors
~~~

#### Restauramos un disco

Para la restauración de un disco del Raid, vamos a meter el cuarto disco `sdd` como reserva, que sería con la etiqueta `spare`, para que este sea sustituido por el disco fallido, que le aplicaremos la etiqueta `FAULTED`.

###### Añadimos un nuevo disco reserva
~~~
zpool add -f Raid1 spare sdd
~~~

###### Comprobamos el estado del nuevo disco con la etiqueta `spares`
~~~
zpool status
      pool: Raid1
     state: ONLINE
      scan: resilvered 278K in 0 days 00:00:04 with 0 errors on Sun Jan 19 18:20:59 2020
    config:

    	NAME        STATE     READ WRITE CKSUM
    	Raid1       ONLINE       0     0     0
    	  mirror-0  ONLINE       0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     ONLINE       0     0     0
    	    sde     ONLINE       0     0     0
    	spares
    	  sdd       AVAIL

    errors: No known data errors
~~~

###### Marcamos fallido un disco
~~~
zpool offline -f Raid1 /dev/sde
~~~

###### Comprobamos el estado del disco fallido con la etiqueta `FAULTED` 
~~~
zpool status
      pool: Raid1
     state: DEGRADED
    status: One or more devices are faulted in response to persistent errors.
    	Sufficient replicas exist for the pool to continue functioning in a
    	degraded state.
    action: Replace the faulted device, or use 'zpool clear' to mark the device
    	repaired.
      scan: resilvered 278K in 0 days 00:00:04 with 0 errors on Sun Jan 19 18:20:59 2020
    config:

    	NAME        STATE     READ WRITE CKSUM
    	Raid1       DEGRADED     0     0     0
    	  mirror-0  DEGRADED     0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     ONLINE       0     0     0
    	    sde     FAULTED      0     0     0  external device fault
    	spares
    	  sdd       AVAIL   

    errors: No known data errors
~~~

###### Reemplazamos el disco fallido por el de reserva
~~~
zpool replace -f Raid1 /dev/sde /dev/sdd
~~~

###### Comprobamos el reemplazo
~~~
zpool status
      pool: Raid1
     state: DEGRADED
    status: One or more devices are faulted in response to persistent errors.
    	Sufficient replicas exist for the pool to continue functioning in a
    	degraded state.
    action: Replace the faulted device, or use 'zpool clear' to mark the device
    	repaired.
      scan: resilvered 345K in 0 days 00:00:03 with 0 errors on Sun Jan 19 18:51:14 2020
    config:

    	NAME         STATE     READ WRITE CKSUM
    	Raid1        DEGRADED     0     0     0
    	  mirror-0   DEGRADED     0     0     0
    	    sdb      ONLINE       0     0     0
    	    sdc      ONLINE       0     0     0
    	    spare-2  DEGRADED     0     0     0
    	      sde    FAULTED      0     0     0  external device fault
    	      sdd    ONLINE       0     0     0
    	spares
    	  sdd        INUSE     currently in use

    errors: No known data errors
~~~

###### Restauramos el disco fallido
~~~
zpool clear Raid1 /dev/sde
~~~

###### Comprobamos el estado de los discos todos con la etiqueta `ONLINE`
~~~
zpool status
      pool: Raid1
     state: ONLINE
      scan: resilvered 45K in 0 days 00:00:03 with 0 errors on Sun Jan 19 18:53:08 2020
    config:

    	NAME        STATE     READ WRITE CKSUM
    	Raid1       ONLINE       0     0     0
    	  mirror-0  ONLINE       0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     ONLINE       0     0     0
    	    sde     ONLINE       0     0     0
    	spares
    	  sdd       AVAIL   

    errors: No known data errors
~~~

Todo este proceso de cambio del disco fallido por el del reserva, se puede automatizar modificando un parametro de zfs.

Para realizar esta modificación es con el parametro `autoreplace`, el cual tenemos que activarlo con `on`.

###### Modificamos configuración para que haga el reemplazo automático
~~~
zpool autoreplace=on Raid1
~~~

###### Comprobamos que lo configurado estan activado
~~~
zpool get all Raid1 | grep autoreplace
    Raid1  autoreplace                    on                             local
~~~

#### Restauramos el Raid

Ahora vamos a restarurar un Raid en caso de error. Para la efectuar la restauracion del Raid, necesitamos crear un checkpoint, el cual se guardará para poder importarlo en un futuro, por defecto solo se puede realizar un checkpoint por pool.

###### Realizamos el checkpoint
~~~
zpool checkpoint Raid1
~~~

###### Comprobamos que se a realizado correctamente el checkpoint
~~~
zpool get all Raid1 | grep checkpoint
    Raid1  checkpoint                     73,5K                          -
    Raid1  feature@zpool_checkpoint       active                         local
~~~

Vamos a hacer que fallen varios discos y cuando restauremos el checkpoint guardado, el Raid volverá a la versión estable en la que le realizamos el checkpoint.

###### Comprobamos el estado del Raid con fallos
~~~
zpool status
      pool: Raid1
     state: ONLINE
      scan: none requested
    checkpoint: created Sun Jan 19 19:53:14 2020, consumes 73,5K
    config:

    	NAME        STATE     READ WRITE CKSUM
    	Raid1       DEGRADED     0     0     0
    	  mirror-0  DEGRADED     0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     FAULTED      0     0     0  external device fault
    	    sde     FAULTED      0     0     0  external device fault
    	spares
    	  sdd       AVAIL   

    errors: No known data errors
~~~

###### Exportamos el Raid para poder añadir el guardado
~~~
zpool export Raid1
~~~

###### Añadimos el Raid guardado con el checkpoint
~~~
zpool import --rewind-to-checkpoint Raid1
~~~

###### Comprobamos el estado dwel Raid
~~~
zpool status
      pool: Raid1
     state: ONLINE
      scan: none requested
    config:
    
    	NAME        STATE     READ WRITE CKSUM
    	Raid1       ONLINE       0     0     0
    	  mirror-0  ONLINE       0     0     0
    	    sdb     ONLINE       0     0     0
    	    sdc     ONLINE       0     0     0
    	    sde     ONLINE       0     0     0
    	spares
    	  sdd       AVAIL   
    
    errors: No known data errors
~~~

Si queremos borrar el checkpoint podemos hacerlo con la opción `-d`.

###### Para borrar el checkpoint
~~~
zpool checkpoint -d pool
~~~

#### Ventajas e inconvenientes respecto al uso de RAID Software con mdadm



--------------------------------------------------------------------
* Realizar ejercicios, con pruebas de funcionamiento, de las principales funcionalidades: compresión, cow, deduplicación, cifrado, etc.
--------------------------------------------------------------------

#### Comprensión

Co

~~~
zfs set compression=lz4 Raid1
~~~

###### Comprobamos 
~~~
zfs get compression Raid1
	NAME   PROPERTY     VALUE     SOURCE
	Raid1  compression  on        local
~~~

###### Además podemos comprobar si esta activado la compresión con `lz4`
~~~
zpool get all Raid1 | grep lz4
	Raid1  feature@lz4_compress           active                         local
~~~

#### Deduplicación

La deduplicación es
Por defecto, la deduplicación esta desactivada:

~~~
zfs get dedup Raid1
~~~

Con el siguiente comando la activamos:

~~~
zfs set dedup=on Raid1
~~~

Comprobamos:

~~~
zfs get dedup Raid1
~~~