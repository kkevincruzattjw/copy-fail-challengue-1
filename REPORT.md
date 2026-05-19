# Reporte Técnico — CVE-2026-31431 "Copy Fail"
## Kevin Cruzate | Introducción a UNIX | UIDE

### 1. El bug raíz

El bug está en `crypto/algif_aead.c`, función `_aead_recvmsg()`. El problema es que se usa `sg_chain()` para encadenar las páginas del TX SGL al RX SGL, y luego se hace `req->src = req->dst`, poniendo páginas del page cache en un scatterlist de escritura. Esto permite escribir en memoria que no debería ser modificable.

### 2. Por qué el write a dst[assoclen + cryptlen] es peligroso

Cuando `req->src == req->dst`, el kernel escribe el resultado de la operación criptográfica sobre las mismas páginas de origen. Esto significa que se pueden escribir 4 bytes controlados en el page cache de cualquier archivo mapeado, incluyendo binarios setuid como `/usr/bin/su`, sin tocar el archivo en disco.

### 3. Por qué el exploit es "stealthy"

El exploit no modifica el archivo en disco. Solo corrompe el page cache en memoria RAM. El inodo del archivo en disco permanece intacto, el timestamp no cambia, el SHA256 del archivo en disco es el mismo. Si haces `ls -la` o `sha256sum` del archivo, parece normal. Solo la versión cargada en memoria está comprometida.

### 4. Conexión con conceptos de clase

- **Page cache**: El kernel mantiene copias de archivos en RAM para acceso rápido. El exploit escribe en esta copia en memoria.
- **setuid**: `/usr/bin/su` tiene el bit setuid activado (`chmod 4755`), lo que significa que se ejecuta con los privilegios del dueño (root), no del usuario que lo llama.
- **Inodos**: El inodo contiene los metadatos del archivo. El exploit no modifica el inodo ni el contenido en disco, solo la copia en el page cache.
- **chmod**: El bit setuid es parte de los permisos del archivo. Sin setuid en `su`, el exploit no funcionaría.

### 5. Lección aprendida

Este bug demuestra cómo múltiples cambios razonables pueden crear una vulnerabilidad grave. La optimización in-place de 2017 tenía sentido por sí sola — evitar copias innecesarias es buena práctica. Pero combinada con AF_ALG y splice(), creó una ruta de escritura en el page cache. Ninguno de los cambios individuales era obviamente peligroso; fue la combinación de ellos la que abrió el hueco de seguridad.

### 6. Problemas encontrados durante el lab

Durante la práctica se encontró que el bzImage oficial del release pesaba 1.4M y causaba Kernel Panic (error -2: No working init found). Se diagnosticó el problema, se compiló el kernel desde fuente (~25 min), y se construyó un initramfs funcional con BusyBox y Python3. Esto permitió completar todos los hitos del laboratorio.
