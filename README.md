# HOMELAB - SERVIDOR LOCAL

## Instalacion y configuracion

1. Instalacion de proxmox VE ISO installer.

2. Activacion de virtualizacion de intel, cambio el orden de arranque para que inicie el usb con proxmox.

   > La virtualizacion es necesaria porque le permite al procesador dividirse de forma eficiente para ejecutar multiples VM a nivel de hardware dividiendo en pequeno fragmentos todos sus nucleos, teniendo una mejor eficiencia en las VM en comparacion de que solo se dividan a nivel de software teniendo un rendimiento casi nativo.

   Si la virtualización está activa en la BIOS, Proxmox podrá usar su tecnología KVM (Kernel-based Virtual Machine) para crear máquinas virtuales. Si no estuviera activa, Proxmox no te dejaría arrancar o no te permitiría crear ninguna máquina virtual dentro del panel.

3. Configuraciones iniciales y definicion de credenciales.

4. Asignacion de direccion ip para el servidor.

   > **hostname:** escribimos el nombre de dominio completo para nuestro servidor ej: proxmox.local

   > **IP Address:** Dejamos en el por defecto o cambiamos el ultimo por un valor mas alto (lo mas recomendable para evitar conflictos de red) y anotamos esa direccion ip ya que con ella accederemos a nuestros servidor desde el navegador.

   > **Gateway:** lo dejamos por defecto.

   > **DNS Server:** podemos dejarlo en el por defecto o cambiarlo por el de Cloudlare `1.1.1.1` o Google `8.8.8.8`.

5. Veremos un resumen con toda la configuracion elegida, Asegurandonos de que la opción Automatically reboot after installation esté marcada si deseamos que se reinicie solo al terminar.

6. Una vez que se reinicie luego de la instalacion veremos la direccion web exacta para entrar al panel de proxmox desde otra pc.

   > El puerto de red por defecto de proxmox es el 8006 para servir su interfaz grafica de administracion.

## Posibles Incovenientes post-instalacion

Luego de instalar todo el sistema operativo el primer inconveniente con el que me encontre fue que al intentar cargar la pagina de proxmox me aparecia "No se puede acceder a este sitio" y es porque la laptop y la pc (donde esta mi servidor) no se estaban comunicando por la red.

### Diagnosticos

**Comando: `ip a`**

Revisa si tu interfaz de red (generalmente llamada eno1, eth0 o enp...) dice UP o DOWN.

Si dice NO-CARRIER o DOWN, significa que el cable Ethernet no está conectado o el puerto no detecta señal. En mi caso mostraba NO-CARRIER que significaba que el puerto de red fisico de la pc no detecta conexion electrica.

**Comando: `ping 192.168..`**

Probe la respuesta por red y como resultado me daba "Host de destino inaccesible" lo que significaba que mi laptop no se llegaba a conectar con mi servidor.

Despues de haber estado investigando informes referentes a estos casos, el motivo exacto de mi inconveniente era que al utilizar usb tp link para conexion a internet a travez de wifi, Proxmox (que esta basado en debian linux) no trae instalado por defecto controladores ni soporte para antenas WIFI USB (como el que yo tenia por el).

Proxmox está diseñado exclusivamente para funcionar mediante puertos de red Ethernet físicos (cable RJ45) y es preferiblemente utilizado de dicha manera debido a que tiene una mejor estabilidad de paquetes.

## Script post-instalacion

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Este es el script de post-instalación para Proxmox VE más popular de la comunidad. Sirve para limpiar, optimizar y dejar listo un servidor Proxmox VE recién instalado sin necesidad de hacer configuraciones manuales complejas.

Al ejecutarlo, abre una interfaz gráfica interactiva en la terminal que realiza automáticamente las siguientes acciones:

- **Corrige los repositorios de paquetes:** Desactiva el repositorio comercial/pago (Enterprise) que viene activado por defecto y activa el repositorio gratuito de la comunidad (No-Subscription).

- **Elimina el aviso de suscripción:** Quita la ventana emergente de "No subscription" (Sin suscripción) que aparece cada vez que inicias sesión en la interfaz web de Proxmox.

- **Actualiza el sistema:** Ejecuta una actualización completa de los paquetes del sistema operativo Debian/Proxmox.

- **Desactiva el repositorio Proxmox Test:** Evita que se instalen paquetes inestables por accidente.

- **Ajustes opcionales de red y sistema:** Desactiva IPv6 (si no lo usas) o aplica optimizaciones de Kernel según tus preferencias.

### Mis configuraciones

- **disable Disable this repo (set Enabled: false):** ya que el repositorio pve-enterprise requiere una licencia o suscripcion de pago de proxmox. Dejarlo habilitado genera errores al intentar actualizar el servidor.

- **disable Comment out (disable) this repo:** Al igual que el repositorio de Proxmox Enterprise anterior, el repositorio ceph-enterprise requiere una suscripción de pago para el almacenamiento distribuido Ceph. Si lo dejas activo sin licencia, dará errores al actualizar los paquetes del sistema.

- **keep Keep as is:** El repositorio pve-no-subscription es el repositorio gratuito de la comunidad. Es completamente esencial mantenerlo habilitado para que tu servidor pueda recibir actualizaciones del sistema sin necesidad de una licencia de pago.

- **yes (Add ceph package sources):** Añade los repositorios gratuitos de Ceph (ceph-tentacle) para evitar problemas de dependencias si decides usar almacenamiento distribuido a futuro.

- **yes (Add (Disabled) pvetest repository):** Agrega la fuente de pruebas en estado inactivo para poder habilitarla manualmente solo si necesitas probar funciones beta sin comprometer la estabilidad actual.

- **yes (Disable subscription nag):** Elimina la ventana emergente de "No valid subscription" cada vez que inicias sesión en la interfaz web.

- **Support Subscriptions Ok:** Pantalla simplemente informativa del autor del script invitando a apoyar el proyecto oficial; no aplica cambios técnicos.

- **yes (Disable high availability):** Detiene los servicios de alta disponibilidad para liberar memoria RAM y CPU, ya que solo son útiles en clústeres de múltiples nodos.

- **yes (Disable Corosync):** Desactiva el servicio de sincronización entre nodos, ahorrando recursos en un entorno de servidor único (standalone).

- **yes (Update Proxmox VE now):** Ejecutó la actualización de todos los paquetes y parches de seguridad utilizando los repositorios gratuitos recién configurados.

- **Post-Install Reminder Ok:** Aviso final recomendando reiniciar el nodo y limpiar la caché del navegador (Ctrl + Shift + R) para aplicar los cambios.

## INSERTAR UN DISCO DE ALMACENAMIENTO EXTRA

Al querer agregar un disco de almacenamiento mas a tus servidor ya sea HDD o SSD te daras cuenta al igual que yo que este no lo reconocera de manera automatica tendras que hacerlo manualmente.

### Paso 1

Identifica el nombre de dispositivo asignado a tu disco duro adicional:

```bash
lsblk -f
```

Busca en la lista la letra del disco (por ejemplo, /dev/sdb o /dev/sdc) y revisa qué sistema de archivos tiene la partición (como ntfs, ext4, fat32, etc.).

### Paso 2

**Montar el Disco: Acceso a carpetas.**

Crea un punto de montaje y accede a las particiones existentes. Reemplaza /dev/sdX1 por el nombre real de tu partición:

```bash
mkdir -p /mnt/mi_disco
mount /dev/sdX1 /mnt/mi_disco
```

Verifica que el disco está montado correctamente enumerando su contenido con:

```bash
ls -la /mnt/mi_disco
```
