[[Win powershell script]]

```bash
#!/bin/bash 
# Función mejorada que INCLUYE liberación 
send_key() { # Presiona la tecla 
echo -ne "\x00\x00$1\x00\x00\x00\x00\x00" > /dev/hidg0 sleep 0.08 # Libera todas las teclas 
echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0 sleep 0.04 # Delay entre teclas (total 0.12s por tecla) 
} 
# 1. Win + R (combinación perfecta) 
echo -ne "\x08\x00\x15\x00\x00\x00\x00\x00" > /dev/hidg0 # Press Win+R
sleep 0.1 echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0

# Release all sleep 0.5 # Espera el cuadro de ejecución

# 2. Escribe "cmd" + Enter 
send_key "\x06" # C 
send_key "\x10" # M 
send_key "\x07" # D 
sleep 0.1 
send_key "\x28" # Enter 
sleep 0.8 # Espera a que cargue cmd 
# 3. Escribe (powershell -ep bypass) + Enter 
for key in "\x13" "\x12" "\x1A" "\x08" "\x15" "\x16" "\x0B" "\x08" "\x0F" "\x0F" "\x2C" "\x2D" "\x08" "\x13" "\x2C" "\x05" "\x1C" "\x13" "\x04" "\x16" "\x16"; do send_key "$key" done 
# Liberación final (redundante por seguridad) 
echo -ne "\x00\x00\x00\x00\x00\x00\x00\x00" > /dev/hidg0 send_key "\x28" sleep 0.8
```