 ### 🔧 **Configuración Limpia desde Cero**

#### 1. **Preparación del Sistema**
```bash
# Actualizar el sistema
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

#### 2. **Configuración Crítica en `/boot/firmware/config.txt`**
```bash
sudo nano /boot/firmware/config.txt
```
**Añade al final** (elimina otras entradas `dwc2` o `dtoverlay` previas):
```ini
# Habilita USB Gadget Mode
dtoverlay=dwc2,dr_mode=peripheral
g_hid.vid=0x1d6b  # Linux Foundation VID
g_hid.pid=0x0104  # Composite Gadget PID
g_hid.device=0     # Forzar reinicio
```

#### 3. **Script de Inicialización (`/usr/local/sbin/hid_init`)**
```bash
sudo nano /usr/local/sbin/hid_init
```
Pega este contenido **mejorado**:
```bash
#!/bin/bash
# Reset completo del controlador USB
echo 0 > /sys/bus/usb/devices/usb1/authorized
echo 1 > /sys/bus/usb/devices/usb1/authorized
sleep 2

# Carga de módulos en ORDEN CRÍTICO
modprobe -r g_hid dwc2 usb_f_hid libcomposite 2>/dev/null
modprobe dwc2 dr_mode=peripheral
modprobe libcomposite
modprobe usb_f_hid
modprobe g_hid

# ConfigFS: Creación manual del gadget
gadget_path="/sys/kernel/config/usb_gadget/g1"
mkdir -p "$gadget_path" || exit 1
cd "$gadget_path" || exit 1

# Identificadores USB
echo 0x1d6b > idVendor
echo 0x0104 > idProduct

# Strings de dispositivo
mkdir -p strings/0x409
echo "fedcba9876543210" > strings/0x409/serialnumber
echo "Raspberry" > strings/0x409/manufacturer
echo "HID Keyboard" > strings/0x409/product

# Función HID
mkdir -p functions/hid.usb0
echo 1 > functions/hid.usb0/protocol    # Keyboard
echo 1 > functions/hid.usb0/subclass    # Boot Interface
echo 8 > functions/hid.usb0/report_length

# Descriptor HID para teclado (hexdump crítico)
echo -ne "\x05\x01\x09\x06\xA1\x01\x05\x07\x19\xE0\x29\xE7\x15\x00\x25\x01\x75\x01\x95\x08\x81\x02\x95\x01\x75\x08\x81\x03\x95\x05\x75\x01\x05\x08\x19\x01\x29\x05\x91\x02\x95\x01\x75\x03\x91\x03\x95\x06\x75\x08\x15\x00\x25\x65\x05\x07\x19\x00\x29\x65\x81\x00\xC0" > functions/hid.usb0/report_desc

# Configuración final
mkdir -p configs/c.1/strings/0x409
echo "Keyboard Config" > configs/c.1/strings/0x409/configuration
ln -s functions/hid.usb0 configs/c.1/

# Asignar controlador USB (método robusto)
udc=$(ls /sys/class/udc/ | head -1)
[ -n "$udc" ] && echo "$udc" > UDC || echo "ERROR: No UDC found" >&2

# Permisos universales
chmod 666 /dev/hidg* 2>/dev/null

# Log de estado
logger -t hid_init "HID Gadget configurado en /dev/hidg0"
```

###### 4. **Habilitar Ejecución Automática**
```bash
sudo chmod +x /usr/local/sbin/hid_init

# Método 1: Crontab (más confiable en Pi Zero)
sudo crontab -e
```
Añade esta línea:
```bash
@reboot /usr/local/sbin/hid_init
```

#### 5. **Creación del script inicial**

```shell
 sudo nano /usr/local/bin/send_win_r.sh
```

Crearemos un script que abra cmd y ejecute powershell, hay que pegar el contenido de [[Win powershell script]]

Una vez realizado esto, damos permisos de ejecución al script

```shell
sudo chmod +x /usr/local/bin/send_win_r.sh
```

#### 6. **Verificación Final**
```bash
sudo reboot

# Tras reiniciar, verifica:
lsmod | grep -E 'g_hid|dwc2'
ls /dev/hidg*
dmesg | tail -20
```

veremos que al usar  tendremos esto
```shell
ls /dev/hidg*
ls: cannot access '/dev/hidg*': No such file or directory
```

No nos preocupemos, debemos ejecutar el siguiente comando

```shell
sudo /usr/local/sbin/hid_init
```

Ejecutémoslo 2 veces para ver que tengamos esto

```shell
sudo /usr/local/sbin/hid_init
/usr/local/sbin/hid_init: line 3: /sys/bus/usb/devices/usb1/authorized: No such file or directory
/usr/local/sbin/hid_init: line 4: /sys/bus/usb/devices/usb1/authorized: No such file or directory
modprobe: ERROR: could not insert 'g_hid': No such device
/usr/local/sbin/hid_init: line 31: echo: write error: Device or resource busy
/usr/local/sbin/hid_init: line 32: echo: write error: Device or resource busy
/usr/local/sbin/hid_init: line 33: echo: write error: Device or resource busy
/usr/local/sbin/hid_init: line 36: echo: write error: Device or resource busy
ln: failed to create symbolic link 'configs/c.1/hid.usb0': File exists
/usr/local/sbin/hid_init: line 45: echo: write error: Device or resource busy
ERROR: No UDC found
```


Cuando tengamos esto, veremos que si ejecutamos `ls /dev/higd*` ahora si tendremos respuesta

```shell
ls /dev/hidg*
|
|- /dev/hidg0

```

Y a partir de aqui ya podemos correr nuestros scripts automatizados haciéndonos pasar por un teclado legitimo

##### Corremos el script

```shell
sudo /usr/local/bin/send_win_r.sh
```



En caso de que no funcione la segunda vez que conectas el dispositivo, realiza lo siguiente

Este error indica que tu script `hid_init` está fallando al intentar configurar el dispositivo USB como un _USB HID gadget_ en tu Raspberry Pi. Vamos paso por paso explicando cada problema y cómo solucionarlo.

---

### 📌 **Errores Clave**

1. **`/sys/bus/usb/devices/usb1/authorized: No such file or directory`**
    
    Esto sugiere que el controlador USB `usb1` no existe o no está activo en ese momento. En Raspberry Pi (usando el puerto OTG, normalmente en la Raspberry Pi Zero, Zero 2 W o 4B), el controlador OTG no se llama `usb1`, y ese path podría ser incorrecto.
    
    ✅ **Solución:** No todos los sistemas tienen `usb1`. Puedes quitar o comentar estas dos líneas:
    
    ```bash
    # echo 0 > /sys/bus/usb/devices/usb1/authorized
    # echo 1 > /sys/bus/usb/devices/usb1/authorized
    ```
    
2. **`modprobe: ERROR: could not insert 'g_hid': No such device`**
    
    Esto significa que el kernel no puede cargar el módulo `g_hid` porque **el hardware o el modo OTG no está correctamente configurado**.
    
    ✅ **Solución:**
    
    - Asegúrate de que el archivo `/boot/config.txt` tiene habilitado el **OTG mode**:
        
        ```shell
        dtoverlay=dwc2
        ```
        
    - Y que en `/boot/cmdline.txt` hayas añadido `modules-load=dwc2` **justo después de `rootwait`**, por ejemplo:
        
        ```shell
        ... rootwait modules-load=dwc2,g_ether ...
        ```
        
    - Reinicia después de hacer estos cambios.
        
3. **`echo: write error: Device or resource busy`**
    
    Esto pasa cuando ya hay una configuración activa y estás tratando de escribir de nuevo en `report_desc`, `UDC`, etc. Básicamente, estás intentando montar el gadget encima de uno ya activo.
    
    ✅ **Solución:** Antes de ejecutar `hid_init`, asegúrate de desmontar gadgets anteriores. Puedes agregar esto al inicio del script:
    
    ```bash
    if [ -d /sys/kernel/config/usb_gadget/g1 ]; then
        echo "" > /sys/kernel/config/usb_gadget/g1/UDC || true
        sleep 1
        rm -rf /sys/kernel/config/usb_gadget/g1
    fi
    ```
    
4. **`ln: failed to create symbolic link 'configs/c.1/hid.usb0': File exists`**
    
    El enlace simbólico ya existe, lo que indica que ejecutaste el script más de una vez sin desmontar el gadget anterior.
    
    ✅ **Solución:** Usa el snippet anterior para limpiar la configuración previa antes de crearla de nuevo.
    
5. **`ERROR: No UDC found`**
    
    Este es el más crítico. Significa que **el kernel no detecta un controlador USB gadget disponible** para usar como HID.
    
    ✅ **Solución:**
    
    - Ejecuta `ls /sys/class/udc/` y verifica que aparece algo como `20980000.usb`
        
        ```bash
        ls /sys/class/udc/
        ```
        
        Si aparece vacío, entonces el kernel no ha cargado `dwc2` correctamente o la placa no permite OTG (por ejemplo, un Raspberry Pi 4 necesita configuraciones especiales en el puerto USB-C).
        

---

### ✅ Recomendaciones Completas

1. Asegúrate de que tu Pi soporta **modo OTG** (Raspberry Pi Zero, Zero W, Pi 4 usando el USB-C con `device-tree` correctamente configurado).
    
2. En `/boot/config.txt`:
    
    ```
    dtoverlay=dwc2
    ```
    
3. En `/boot/cmdline.txt`, añade:
    
    ```
    modules-load=dwc2,libcomposite
    ```
    
    (asegúrate de no romper la línea: **toda debe estar en una sola línea**).
    
4. Reinicia.
    

---

### 🛠️ Alternativa rápida para probar el UDC

Antes de ejecutar tu script, asegúrate de que el gadget framework está disponible:

```bash
modprobe libcomposite
ls /sys/class/udc/
```

Si esto no muestra nada, el UDC (USB Device Controller) no está funcionando — sin esto, no puedes crear gadgets.