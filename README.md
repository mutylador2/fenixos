# FenixOS 1.3 «Titán»

**Sistema operativo x86_64 completo construido desde cero** — sin Linux, sin GRUB, sin código ajeno: bootloader propio en ensamblador con **menú multiboot**, kernel 64-bit en C, pila TCP/IP propia, usuarios y roles, **instalador gráfico que convive con Windows/Linux**, interfaz gráfica, navegador web y lenguaje de programación interno. Arquitectura por capas inspirada en el diseño de Windows (HAL → Kernel → Executive → Subsistemas) con identidad y diseño totalmente propios.

![Información del sistema](capturas/33-sistema-info.png)

---

## 1. Qué es FenixOS

FenixOS es un sistema operativo educativo-funcional que arranca **directamente del sector de arranque de un disco o CD**, sin cargar ningún sistema existente. Todo lo que ves —la transición a 64 bits, la pantalla gráfica, la multitarea, la red, el ratón, la terminal— es código propio que vive en una imagen de 64 MB (disco) o en un CD booteable.

| Componente | Descripción |
|---|---|
| **Bootloader** | Stage1 (512 bytes MBR, LBA+fallback CHS) + Stage2: e820, **menú multiboot con 5 opciones y chainload**, VBE 1024×768×32, tablas de página y salto a modo largo. Stub El Torito sin emulación para CD + **MBR híbrido para USB** |
| **Kernel 64-bit** | GDT/IDT propias, PIC remapeado, PIT a 100 Hz, PMM (bitmap sobre e820), heap de 16 MB, planificador round-robin con estadísticas de CPU |
| **Red TCP/IP** | Driver e1000 (MMIO, por sondeo), ARP, IPv4, ICMP (ping), UDP, DNS, TCP minimal y HTTP/1.1 — con sockets y navegador **FenixNav** |
| **Almacenamiento** | Capa de bloques unificada con drivers **ATA PIO** y **AHCI/SATA** (canal PCI 0106), particiones MBR y zona de instalación en partición propia (tipo 0x63) |
| **Instalador gráfico** | FenixInstall: elige disco, muestra particiones existentes, instala en **disco completo** o **junto a otros OS** (crea partición propia sin tocar nada) |
| **Usuarios y roles** | Tres roles jerárquicos (invitado/usuario/admin), login con hash djb2, permisos por orden, cuentas y **claves persistentes en disco** (`useradd`) |
| **Configuración persistente** | Tabla clave=valor respaldada **en disco** (ATA/AHCI, LBA reservado) + **menú multiboot persistente** (opción y autoarranque) + reflejo en el RAMFS |
| **Registro de eventos** | Anillo en RAM de 96 eventos con niveles INFO/WARN/ERROR, consultable con `logs` y exportable al RAMFS |
| **Panel del sistema** | Ventana **Información del sistema** (`sistema`): RAM total/usable, CPU (marca, cores, freq), GPU (VBE + PCI), discos, red, arranque y OS |
| **FenixScript** | Lenguaje BASIC-like para **programar dentro del sistema**: los scripts `.fx` son extensiones compartibles |
| **RAMFS** | 32 archivos × 4 KB, precargado con documentación y extensiones de ejemplo |
| **Apps gráficas** | Editor de texto, Calculadora con botones, Visor de archivos con scroll, FenixNav, FenixDemo, Información del sistema, Instalador, Acerca de |
| **FenixCompat** | Análisis de ejecutables Windows MZ/PE (máquina, subsistema, punto de entrada) |
| **Drivers** | Teclado PS/2, ratón PS/2, RTC CMOS, altavoz PC, COM1, **ATA PIO y AHCI/SATA**, **NIC e1000**, enumeración PCI y detección de controladoras USB |

### Arquitectura por capas (estilo Windows, identidad propia)

```
┌────────────────────────────────────────────────────────────┐
│  SUBSISTEMAS   FenixShell │ FenixNav │ Calculadora │       │
│                Editor │ Visor │ RAMFS │ FenixScript │      │
│                FenixCompat │ Logs │ Sonido │ FenixDemo │   │
│                Información del sistema │ Instalador        │
├────────────────────────────────────────────────────────────┤
│  EXECUTIVE     Gestor de ventanas │ Compositor │ Usuarios  │
│                Configuración │ Planificador │ Entrada      │
├────────────────────────────────────────────────────────────┤
│  RED           HTTP/1.1 │ DNS │ TCP minimal │ UDP │        │
│                ICMP │ IPv4 │ ARP │ Ethernet                │
├────────────────────────────────────────────────────────────┤
│  KERNEL        PMM (e820+bitmap) │ Heap │ Tareas │ GDT │   │
│                IDT │ Dispatcher │ Panic │ klog (anillo)    │
├────────────────────────────────────────────────────────────┤
│  DRIVERS       e1000 (MMIO) │ ATA PIO │ AHCI/SATA │ BLK    │
│                PS/2 teclado+ratón │ RTC │ PIT │ PIC │      │
│                Altavoz │ COM1 │ PCI/USB                     │
├────────────────────────────────────────────────────────────┤
│  HAL           VBE framebuffer (doble búfer) │ fuente 8×16 │
├────────────────────────────────────────────────────────────┤
│  BOOTLOADER    Stage1 (MBR) → MENÚ MULTIBOOT → Stage2 →    │
│                modo largo. Stage1CD (El Torito) + MBR      │
│                híbrido (USB) + chainload de otros OS       │
└────────────────────────────────────────────────────────────┘
```

Detalles técnicos consultables en el código fuente:

- **Modo largo real**: la CPU entra en IA-32e (LME + paginación de 4 niveles) antes de ejecutar una sola línea de C.
- **Identidad mapeada**: primer gigabyte con páginas de 2 MB + todo el GB del framebuffer y del MMIO de la NIC; dirección física == virtual.
- **Multitarea**: 8 tareas máximo, pilas de kernel de 16 KB, planificación preemptive round-robin desde la IRQ0 con contador de cambios por tarea (`ps` muestra el % de CPU real de cada una).
- **Pila TCP sin interrupciones**: la NIC funciona por sondeo (polling), lo que evita acoplarla al IDT; TCP implementa SYN/ACK, PSH, FIN, RST, ventana de 16 KB y reintentos.
- **Arranque cuádruple**: el mismo kernel arranca desde disco (LBA o CHS), desde VHD (VirtualBox), desde CD (blob El Torito de hasta 230 KB copiado a RAM por la BIOS) y **desde USB grabado con Rufus/dd** (MBR híbrido dentro del propio ISO).

---

## 2. Contenido del paquete

| Archivo | Uso |
|---|---|
| **fenixos.iso** | **ISO híbrida**: CD booteable (El Torito) y a la vez imagen USB booteable (Rufus modo DD / `dd`). La forma recomendada de probar |
| **fenixos.vhd** | Disco duro virtual fijo de 64 MB para VirtualBox |
| **README.md** | Este manual |
| **capturas/** | Capturas reales del sistema en funcionamiento (38 verificaciones) |
| **fenixos-codigo-fuente.zip** | Código completo (boot, kernel, scripts de build y pruebas) |
| **instalar/** | Guía y utilidades para **hardware real** (sección 10) |

> A partir de la v1.2 la imagen cruda `fenixos.img` ya no se incluye (se puede regenerar desde el fuente con `make`).

---

## 3. Puesta en marcha

### QEMU (más rápido para probar)

```bash
# desde CD (recomendado)
qemu-system-x86_64 -cdrom fenixos.iso -m 256 -vga std

# con red (necesaria para ping/nav)
qemu-system-x86_64 -cdrom fenixos.iso -m 256 -vga std -nic user,model=e1000

# desde disco
qemu-system-x86_64 -drive format=raw,file=fenixos.img -m 256 -vga std -nic user,model=e1000
```

> Si compilas tú mismo, el `make qemu` del fuente ya arranca con red.

### VirtualBox

1. Máquina nueva → tipo *Other / Other 64-bit* (o DOS/Other), **256 MB** de RAM.
2. **Opción A (CD)**: Almacenamiento → controlador IDE → añade `fenixos.iso` como CD/DVD → arranca.
3. **Opción B (VHD)**: Almacenamiento → añade `fenixos.vhd` como disco SATA/IDE → arranca.
4. **Red (opcional)**: Configuración → Red → Adaptador 1 → *NAT* + **Tipo: Intel PRO/1000 MT Desktop** (la e1000 de VirtualBox). Con eso `ping` y `nav` funcionan igual que en QEMU.

### Hardware real (resumen — guía completa en la sección 10)

Graba `fenixos.iso` en un USB con **Rufus (modo DD)** o `dd` y arranca el equipo desde él (modo Legacy/CSM): verás el menú multiboot. El sistema es 100 % autónomo.

---

## 4. Menú multiboot y convivencia con otros sistemas

Al arrancar desde disco o USB aparece el **gestor de arranque de FenixOS**:

```text
  FenixOS 1.3 'Titán' - Gestor de Arranque
  1) Iniciar FenixOS
  2) FenixOS - modo seguro (sin red ni ratón)
  3) Iniciar otro sistema de este disco
  4) Iniciar desde otro disco (USB / 2do disco)
  5) Información del sistema
  Autoarranque en 8  -  opción por defecto: 1 (pulsa 1-5)
```

- **Opción 3 (chainload)**: lista las particiones del disco (Windows NTFS, Linux, FAT…) y transfiere el control a su sector de arranque. **Windows y Linux siguen arrancando igual**: FenixOS no modifica ni un byte de las particiones existentes.
- **Opción 4**: arranca del primer/segundo disco del BIOS (USB, segundo HDD).
- **Opción 2 (modo seguro)**: arranque a 800×600, sin red, sin ratón y sin sonido — útil para depurar.
- **Opción 5**: pantalla de información del hardware en modo texto.
- La opción por defecto y el tiempo de autoarranque son **persistentes**: `arranque 2 4` deja el modo seguro como arranque por defecto con 4 segundos.

```text
arranque              # ver la configuración actual del menú
arranque 1 8          # opción por defecto 1, autoarranque en 8 s
arranque 2 4          # arrancar en modo seguro por defecto, 4 s
```

### Instalador gráfico: FenixOS junto a Windows/Linux

El **instalador gráfico** (`instalador` o el icono del escritorio) instala FenixOS en otro disco **sin destruir nada**:

1. Elige el disco de destino en la lista (discos ATA/AHCI detectados, con su tamaño y particiones).
2. El instalador muestra las particiones existentes de cada disco.
3. Elige el modo:
   - **Disco completo**: dedica el disco entero a FenixOS (con confirmación).
   - **Junto a otros OS** (recomendado): crea una **partición nueva tipo 0x63** en el espacio libre, escribe el MBR respetando las particiones existentes e instala stage1/stage2/kernel/config dentro de su partición. **Windows y Linux quedan intactos y arrancables** (opción 3 del menú).
4. **Confirmar instalación**: escribe todo y muestra el resultado.

El procedimiento está verificado automáticamente: tras instalar junto a un «Windows» (NTFS) y un «Linux» simulados, ambas particiones conservan sus datos byte a byte y el equipo arranca tanto FenixOS (partición 0x63) como el otro sistema (chainload).

---

## 5. Sesión, usuarios y roles

El sistema arranca con la sesión de la clave `autologin` (por defecto **admin**). Cuentas de fábrica:

| Usuario | Clave | Rol | Puede |
|---|---|---|---|
| `admin` | `fenix` | admin | Todo: configuración, apagar, usuarios, logs… |
| `usuario` | `1234` | usuario | Archivos, apps, red, scripts, kill de tareas |
| `invitado` | (sin clave) | invitado | Solo lectura: ayuda, info, mem, archivos, leer, logs… |

```text
login admin fenix        # cambiar de sesión
login usuario 1234
quiensoy                 # usuario y rol actuales
usuarios                 # listado de cuentas (solo admin)
salir                    # cerrar sesión (pasa a invitado)
useradd ana 5678 1       # crear cuenta «ana», rol 1=usuario (persistente)
```

Las cuentas creadas con `useradd` (nuevo en 1.3) **se guardan en disco y sobreviven al reinicio** — verificado automáticamente (crear `pepe`, reiniciar, hacer login). Las ordenes protegidas responden con `permiso denegado` y quedan registradas en el log de seguridad. Ejemplo real: con sesión `usuario`, `apagar` es rechazado.

---

## 6. Red TCP/IP y navegador FenixNav

Con la NIC e1000 presente (QEMU `-nic user,model=e1000` o VirtualBox con Intel PRO/1000 MT), el kernel levanta su pila con las direcciones estándar de NAT:

- IP `10.0.2.15/24`, puerta `10.0.2.2`, DNS `10.0.2.3` (configurables y **persistentes**).

```text
net                      # estado: MAC, IP, GW, DNS, enlaces, tramas TX/RX
net ip 10.0.2.15 10.0.2.2 10.0.2.3   # reconfigurar
ping 10.0.2.2            # 4 ecos ICMP reales con RTT
ping example.com         # resuelve por DNS y luego hace ping
nav http://example.com   # descarga la página y la muestra en el visor
```

FenixNav hace DNS → TCP → HTTP/1.1 GET de verdad (probado con `example.com` a través del NAT de QEMU/VirtualBox), convierte el HTML a texto (títulos, listas, entidades, saltos por etiquetas de bloque) y lo muestra en una ventana con desplazamiento con las flechas ↑/↓.

**Límites honestos de la v1.3**: sin TLS (solo `http://`), un socket TCP activo, sin fragmentación IP, sin DHCP (IP estática), sin retransmisión agresiva. La API de sockets existe en el kernel (`net_tcp_connect/send/recv/close`) para futuras apps.

---

## 7. Aplicaciones gráficas

| App | Orden | Descripción |
|---|---|---|
| **Calculadora** | `calculadora` | Ventana con teclado numérico clicable (ratón) y soporte de teclado: `5 + 3 =` |
| **Visor** | `visor archivo` | Muestra archivos del RAMFS con scroll (↑/↓, Inicio/Fin); también lo usa FenixNav |
| **Editor** | `editor archivo` | Editor de texto con Guardar/Limpiar; escribe en el RAMFS |
| **FenixDemo** | `demo on/off` | Ventana animada de demostración |
| **Información del sistema** | `sistema` | Panel con RAM, CPU, GPU, discos, red, arranque y versiones |
| **Instalador** | `instalador` | Instalación gráfica en otro disco (sección 4) |
| **Acerca** | `acerca` | Información del sistema |

El escritorio tiene **8 iconos** (Terminal, Demo, Editor, Calculadora, Visor, Sistema, Instalador, Acerca) — un clic abre cada app. Las ventanas se arrastran con el ratón y la barra de tareas muestra las abiertas.

---

## 8. FenixScript: programar dentro del sistema

Los archivos `.fx` son scripts numerados estilo BASIC que se ejecutan con `fx`. Sirven como **extensiones del sistema creadas por cualquier usuario**: se escriben con el editor (o `escribir`), se comparten copiando el texto, y `ext` las lista con su autor/versión.

```text
#ext: Autor | 1.0 | descripcion de la extension
10 PRINT Hola desde FenixScript! n = $n
20 LET n = 3
30 LET n = n * 2 + 1
40 IF n > 6 GOTO 60
50 PRINT nunca sale
60 EXEC echo orden de la shell desde el script
70 FIN
```

Instrucciones: `PRINT` (con `$variables`), `LET var = expresión` (+ - x / encadenados, variables con o sin `$`), `IF a op b GOTO n` (`== != < > <= >=`), `GOTO n`, `EXEC <orden shell>`, `FIN`, comentarios con `#`. Límite de seguridad: 5000 pasos (evita bucles infinitos).

Extensiones de ejemplo incluidas: `hola.fx`, `cuenta.fx`, `sistema.fx`.

---

## 9. Registro de eventos y configuración persistente

### logs

```text
logs              # últimos 96 eventos con nivel y segundo de arranque
logs guardar      # exporta el registro al RAMFS (sistema.log)
logs limpiar      # vacía el anillo
```

Todo evento relevante queda registrado: arranque por capas, sesiones (`login`), órdenes ejecutadas, denegaciones de permiso, ecos ICMP, descargas HTTP, escrituras de configuración, tareas creadas/terminadas, divisiones entre cero de la calculadora… y los pánicos.

### config

```text
config                    # listar claves activas
config hostname fenix42   # fijar una clave (admin)
tema azul                 # (los temas también se registran)
config guardar            # ESCRIBIR EN DISCO (LBA reservado, ATA PIO)
config cargar             # releer del disco
```

Al arrancar, el kernel lee el sector de configuración del disco y aplica **hostname, tema, autologin y las IPs de red** antes de mostrar el escritorio. Si el disco no existe (arranque solo-CD) usa valores por defecto y lo indica en el log.

---

## 10. Instalación en hardware real

### A. USB booteable con Rufus (Windows) o dd (Linux/macOS) — recomendado

La **misma `fenixos.iso` sirve para CD y para USB**: contiene un MBR híbrido que el BIOS arranca como disco.

1. **Rufus**: selecciona `fenixos.iso`, esquema *MBR*, sistema destino *BIOS (o UEFI-CSM)* y, si pregunta, elige **modo DD** («escribir imagen exacta»). Rufus avisará de que el ISO es híbrido: es lo esperado.
2. **Linux/macOS**: `sudo dd if=fenixos.iso of=/dev/sdX bs=4M conv=fsync status=progress`
3. Arranca el PC desde el USB con **modo Legacy/CSM** activado: verás el **menú multiboot** (con la opción 3 puedes arrancar el Windows/Linux ya instalado en ese disco, y el instalador gráfico puede instalar FenixOS junto a él sin tocarlo).

El script `instalar/instalar-usb.sh` hace el equivalente a Rufus desde Linux con confirmación segura.

### B. Disco duro secundario / VHD

- Escribe la imagen en un disco SATA/IDE dedicado (`dd` igual que arriba) y arranca desde él.
- `fenixos.vhd` puede convertirse a formato físico con `qemu-img convert` o adjuntarse tal cual a VirtualBox.

### C. CD/DVD físico

Graba `fenixos.iso` con cualquier grabadora (Brasero, ImgBurn, `wodim`). Arranca en cualquier PC con BIOS legacy.

**Requisitos de hardware**: CPU x86_64, BIOS legacy con VBE 2.0+ (cualquier PC post-2004), 128 MB RAM, VGA compatible. Probado en QEMU y diseñado para hardware común: controladoras ATA en modo compatibility, PS/2 o emulación USB de teclado/ratón activada en la BIOS.

> Nota: el modo de vídeo y los sectores se negocian en el arranque; si tu equipo es muy moderno (solo UEFI puro), usa la opción CSM del firmware.

---

## 11. Compatibilidad con Windows y USB

**FenixCompat (`win archivo.exe`)** analiza ejecutables MZ/PE: valida la cabecera DOS, localiza la cabecera PE, e informa de máquina destino (i386/x64/ARM), secciones, punto de entrada, image base y subsistema (GUI/consola). Incluye `muestra.exe` de prueba. **FenixOS no ejecuta binarios de Windows**: eso exigiría una capa Win32 completa (emulador x86 + traducción de syscalls), documentada como ruta futura en el fuente.

**USB**: se detectan y listan las controladoras del bus PCI (UHCI/OHCI/EHCI/xHCI) con `usb`. No hay pila host funcional todavía: teclados/ratones USB dependen de la emulación PS/2 de la BIOS/VM, y el **anclaje de red del móvil por cable (RNDIS/NCM) para compartir Internet** requiere dicha pila + driver RNDIS — está en la hoja de ruta junto a almacenamiento USB.

---

## 12. FenixShell: referencia de órdenes

| Orden | Descripción | Rol mínimo |
|---|---|---|
| `ayuda` | Lista completa de órdenes | invitado |
| `info` | Ficha del sistema estilo neofetch | invitado |
| `mem` / `ps` / `uptime` | Memoria · tareas con % CPU · tiempo encendido | invitado |
| `quiensoy` / `login u c` / `salir` | Sesión y cambio de usuario | invitado |
| `usuarios` / `useradd n c r` | Listado / creación de cuentas persistentes | admin |
| `archivos` / `leer` / `tocar` / `borrar` | RAMFS: listar, leer, crear, borrar | invitado / usuario |
| `escribir a txt` | Crear/escribir un archivo | usuario |
| `editor [a]` / `visor [a]` | Editor / visor gráficos | usuario |
| `calculadora` / `calc a + b` | Calculadora gráfica / de terminal | usuario |
| `fx script.fx` / `ext` | Ejecutar extensión / listar extensiones | usuario |
| `win archivo.exe` | FenixCompat: análisis MZ/PE | usuario |
| `ping host` / `net` / `nav url` | Red: ecos, estado, navegador | usuario |
| `logs [guardar/limpiar]` | Registro de eventos | invitado / admin |
| `config` / `config guardar` | Ver/editar/persistir configuración | ver: todos, editar: admin |
| `disco` / `usb` | Discos ATA/AHCI y zona de instalación / controladoras USB | invitado |
| `sistema` | Panel de información del hardware (RAM/CPU/GPU/discos) | invitado |
| `instalador` | Instalación gráfica en otro disco | admin |
| `arranque [op s]` | Menú multiboot: opción y autoarranque persistentes | admin |
| `tema fuego|azul|bosque|vapor` | Tema del escritorio (se recuerda) | usuario |
| `sonido` / `musica` / `demo on/off` | Altavoz PC / melodía / demo animada | usuario |
| `fecha` / `raton` / `echo` / `limpiar` / `fenix` | Utilidades varias | invitado |
| `kill id` / `reboot` / `apagar` | Terminar tarea / reiniciar / apagar | usuario / admin |

Alias tipo Unix: `ls cat touch rm write clear whoami date mouse about play disk dmesg help`.

---

## 13. Estructura del código fuente

```
fenixos/
├── boot/          stage1.asm (MBR 512 B), stage2.asm (menú multiboot + chainload),
│                  stage1cd.asm (El Torito), stage1hybrid.asm (MBR USB/Rufus)
├── kernel/        entrada en C (kmain) + módulos por capa:
│   ├── núcleo:    gdt idt isr pic pit pmm heap task string printf klog serial
│   ├── drivers:   kbd mouse rtc sound ata ahci blk e1000 pci usb power
│   ├── red:       net (ARP/IP/ICMP/UDP/DNS/TCP/HTTP)
│   ├── executive: fb gui config users sysinfo installer
│   └── apps:      shell ramfs fx (FenixScript) win (FenixCompat) font console
│                  (kernel/embed: stage1+stage2 incrustados para el instalador)
├── scripts/       build_image.py build_iso.py + suites de prueba (QEMU headless)
└── Makefile       make → fenixos.img + fenixos.vhd + fenixos.iso
```

Compilación: `gcc` freestanding (`-mgeneral-regs-only`, sin red-zone) + `nasm`. Sin libc ni dependencias externas.

### Pruebas automatizadas

| Suite | Cubre | Estado |
|---|---|---|
| `test_v13.py` | 35 comprobaciones: AHCI/SATA con 2 discos, panel `sistema`, `useradd` persistente, configuración del menú multiboot, **instalador gráfico por clicks**, verificación binaria del disco instalado (particiones ajenas intactas), arranque desde partición 0x63, **chainload al otro OS**, modo seguro autoarrancado y usuarios tras reinicio | **35/35** |
| `test_usb.py` | Simula Rufus modo DD: vuelca el ISO a disco, comprueba el MBR híbrido y arranca (menú → escritorio) | OK |
| `test_v12.py` | 22 comprobaciones: arranque, red (ping real + HTTP real a example.com), calculadora por clicks, FenixScript, FenixCompat, permisos, logs y **persistencia tras reinicio** | **22/22** |
| `test_mouse.py` | 10 comprobaciones de ratón: movimiento, clicks, arrastre | **10/10** |
| `test_v11.py` | 9 comprobaciones: RAMFS, editor con guardado, sonido | **9/9** |
| `test_qemu.py` / `smoke_final.py` | Regresión del escritorio / arranque por CD y VHD fresco | 6/6 · OK ×2 |

---

## 14. Historial de versiones

- **1.3 «Titán»** — **menú multiboot con chainload** (convivencia real con Windows/Linux: opción 3/4, autoarranque y opción por defecto persistentes con `arranque`); **driver AHCI/SATA** + capa de bloques unificada con detección de particiones; **instalador gráfico** (disco completo / «junto a otros OS» que crea una partición 0x63 sin tocar nada); **panel de información del sistema** (`sistema`); cuentas de usuario persistentes (`useradd` + login tras reinicio); **ISO híbrida CD+USB** (MBR propio para Rufus/dd con el menú multiboot); modo seguro a 800×600; IDENTIFY ATA con reintentos; 8 iconos.
- **1.2 «Horizonte»** — pila TCP/IP propia con sockets y navegador FenixNav; usuarios/roles/permisos; registro de eventos; configuración persistente en disco (ATA); calculadora y visor gráficos; FenixScript y extensiones `.fx`; FenixCompat (MZ/PE); detección PCI/USB; RAMFS ampliado (32×4 KB); 6 iconos; `%CPU` por tarea; guía de instalación en hardware real.
- **1.1 «Renacer»** — RAMFS con órdenes de archivos, editor gráfico con guardado, altavoz PC con melodía, arranque por CD (El Torito sin emulación) y entrega `.iso` + `.vhd`.
- **1.0.1** — captura de ratón PS/2 corregida (prefijo 0xD4, ACK, IRQ12) y orden `raton`.
- **1.0** — primera versión completa: bootloader, kernel 64-bit, GUI con ventanas, FenixShell.

---

## 15. Créditos y licencia

Sistema operativo de juguete-serio construido con fines educativos: bootloader, kernel, drivers, pila de red, GUI, shell, lenguaje de scripts y herramientas de prueba escritos íntegramente para este proyecto. Usa, estudia, modifica y comparte.
