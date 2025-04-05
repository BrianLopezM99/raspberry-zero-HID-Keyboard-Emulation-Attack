¡Excelente idea! Vamos a reconstruir desde cero **la configuración definitiva** para el HID Keyboard en Raspberry Pi Zero 2 W, integrando todo lo aprendido. Sigue este proceso paso a paso:

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
