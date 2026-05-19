# Copy Fail — CVE-2026-31431 | Kevin Cruzate

## Problemas encontrados y soluciones

### 1. Kernel Panic: "Failed to execute /init (error -2)"
El bzImage oficial del release (1.4M) causaba kernel panic al arrancar QEMU.
**Diagnóstico:** El kernel minimalista del release no era compatible con el initramfs generado por el lab.
**Solución:** Se compiló el kernel desde fuente (`make kernel`, ~25 min) generando un bzImage funcional de 15M, y se reconstruyó el initramfs con BusyBox + Python3.

### 2. Python sin librerías en la VM
El initramfs inicial no incluía las librerías de Python necesarias para el exploit.
**Solución:** Se copiaron las librerías de Python3 del host al initramfs antes de empaquetarlo.

### 3. Binario `su` sin librerías compartidas
El exploit fallaba con errores de `libpam.so.0` y `libaudit.so.1`.
**Solución:** Se identificaron todas las dependencias con `ldd $(which su)` y se copiaron al initramfs.

### 4. PAM no configurado
El binario `su` abortaba con "Critical error" por falta de configuración PAM.
**Solución:** Se creó `/etc/pam.d/su` con `pam_permit.so` para el entorno de laboratorio.

### 5. Proceso QEMU bloqueando la terminal
QEMU bloqueaba la terminal al paniquear, impidiendo nuevos comandos.
**Solución:** `pkill -9 -f qemu` desde una terminal nueva para liberar el proceso.

## Resultado final
- Hito 1: 2.0/2.0
- Hito 2: 3.0/3.0  
- Hito 3: 1.2/1.5
- Hito 4: 2.0/2.0
- Bonus:  0.5/0.5
- **Total: 8.70/9.0 (97%)**
