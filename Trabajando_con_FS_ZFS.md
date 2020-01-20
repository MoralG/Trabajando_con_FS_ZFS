# Sistemas de ficheros avanzados ZFS

### Creación del escenario

Vamos a crear un escenario que incluya una máquina y varios discos asociados a ella de 500MB.

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

Tenemos que compilar el paquete. Para compilarlo tenemos que descargar las dependencias, clonar un repositorio de Github, donde esta alojado dicho paquete y lugo empezar la compilación.

~~~
apt-get install build-essential autoconf automake libtool gawk alien fakeroot ksh
apt-get install zlib1g-dev uuid-dev libattr1-dev libblkid-dev libselinux-dev libudev-dev
apt-get install libacl1-dev libaio-dev libdevmapper-dev libssl-dev libelf-dev
apt-get install linux-headers-$(uname -r)
~~~

~~~
wget https://github.com/zfsonlinux/zfs/releases/download/zfs-0.8.2/zfs-0.8.2.tar.gz
~~~

~~~
tar axf zfs-0.8.2.tar.gz
cd zfs-0.8.2
chown -R root:root ./
chmod -R o-rwx ./
~~~

~~~
./configure \
    --disable-systemd \
    --enable-sysvinit \
    --disable-debug \
    --with-spec=generic \
    --with-linux=$(ls -1dtr /usr/src/linux-headers-*.*.*-common | tail -n 1) \
    --with-linux-obj=$(ls -1dtr /usr/src/linux-headers-*.*.*-amd64 | tail -n 1)
~~~

~~~
make -j1 && make install
~~~

~~~
cd /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-import /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-mount /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-share /etc/init.d/
ln -s /usr/local/etc/init.d/zfs-zed /etc/init.d/
~~~

~~~
update-rc.d zfs-import defaults
update-rc.d zfs-mount defaults
update-rc.d zfs-share defaults
update-rc.d zfs-zed defaults
~~~

~~~
modprobe zfs
~~~

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

### Gestiona los discos adicionales con ZFS


* Configura los discos en RAID, haciendo pruebas de fallo de algún disco y sustitución, restauración del RAID. Comenta ventajas e inconvenientes respecto al uso de RAID software con mdadm

#### Creamos un Raid 1

###### Creamos un nuevo Pool con Raid 1
~~~
zpool create -f Raid1 mirror /dev/sdb /dev/sdc /dev/sde
~~~

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

###### Añadimos un nuevo disco reserva
~~~
zpool add -f Raid1 spare sdd
~~~

###### Comprovamos el estado del nuevo disco con la etiqueta `spares`
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

~~~
zpool checkpoint Raid1
~~~

###### Creamos un ckeckpoint del Raid
~~~
zpool get all Raid1 | grep checkpoint
~~~

###### Comprobamos que se a realizado correctamente el checkpoint
~~~
zpool get all Raid1 | grep checkpoint
    Raid1  checkpoint                     73,5K                          -
    Raid1  feature@zpool_checkpoint       active                         local
~~~

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

###### Para borrar el checkpoint
~~~
zpool checkpoint -d pool
~~~

* Realiza ejercicios con pruebas de funcionamiento de las principales funcionalidades: compresión, cow, deduplicación, cifrado, etc.

~~~
zfs set compression=lz4 Raid1
~~~