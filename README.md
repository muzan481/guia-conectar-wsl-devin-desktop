# Cómo conectar Devin Desktop a WSL (Ubuntu) sin morir en el intento

Guía sencillita, paso a paso, de cómo conectar **Devin Desktop** a tu **WSL** (igual que haces en VS Code con "Remote - WSL") y qué hacer si te sale este error:

```
Could not establish connection to WSL distro "Ubuntu"

[Error] Error resolving authority
Error: Couldn't install vscode server on remote server, install script returned non-zero exit status
```

---

## ¿Qué está pasando, en cristiano?

Cuando Devin Desktop se conecta a WSL hace lo mismo que VS Code: **descarga e instala un "servidor" dentro de tu Ubuntu** (en la carpeta `~/.devin-server`). Ese servidor es el que hace que puedas abrir carpetas, usar la terminal y correr el agente de Devin dentro de Linux.

El error de arriba significa una sola cosa: **esa instalación falló**. Normalmente es por una de estas tres razones:

1. **WSL no tiene internet** (muy típico si usas VPN, Zscaler, Cloudflare WARP, Tailscale, GlobalProtect, etc.).
2. **Quedó una instalación a medias** de un intento anterior (archivos rotos o un "candado" `.installation_lock` olvidado).
3. **Fue un fallo puntual** (se cortó la descarga, WSL estaba arrancando, etc.). En mi caso fue esto: con darle a **Retry** un rato después, funcionó.

Vamos a revisar en orden, de lo más fácil a lo más "heavy".

---

## Paso 0: Darle a "Retry"

Parece obvio, pero es lo primero. En la ventana del error pulsa **Retry**. Si conecta, listo, no sigas leyendo.

Si no, sigue.

---

## Paso 1: Revisar que WSL esté bien

Abre **PowerShell** (o la terminal de Windows) y escribe:

```powershell
wsl --version
wsl -l -v
```

Lo que tienes que ver:

- Una versión de WSL **2.0 o más nueva** (yo tenía la 3.0.3).
- Tu distro `Ubuntu` con **VERSION 2**.

Si tu WSL es viejo, actualízalo:

```powershell
wsl --update
```

---

## Paso 2: Comprobar si WSL tiene internet

Sigue en PowerShell y ejecuta:

```powershell
wsl -d Ubuntu -- bash -lc "curl -sS -o /dev/null -w '%{http_code}\n' -m 15 https://windsurf-stable.codeiumdata.com/ ; curl -sS -o /dev/null -w '%{http_code}\n' -m 15 https://google.com"
```

¿Cómo leer el resultado?

- **Salen dos números** (por ejemplo `404` y `301`): WSL tiene internet. (Sí, el `404` está bien: solo queríamos ver que el servidor responde.) Salta al **Paso 4**.
- **Se queda pensando y sale un error tipo `timed out` o `Could not resolve host`**: WSL no tiene internet. Ve al **Paso 3**.

---

## Paso 3 (solo si WSL no tiene internet): activar la red "mirrored"

Por defecto WSL usa una red interna (NAT) que muchas VPN no dejan salir. La solución es decirle a WSL que use la misma red que Windows.

1. Abre el Bloc de notas y crea (o edita) este archivo:

   ```
   C:\Users\TU_USUARIO\.wslconfig
   ```

   > Ojo: el archivo se llama literalmente `.wslconfig` (con el punto delante y sin `.txt` al final).

2. Pega esto dentro y guarda:

   ```ini
   [wsl2]
   networkingMode=mirrored
   dnsTunneling=true
   autoProxy=true
   ```

3. Reinicia WSL desde PowerShell:

   ```powershell
   wsl --shutdown
   ```

4. Vuelve a probar el **Paso 2**. Ahora deberían salir los números.

**Plan B:** si no puedes tocar ese archivo, apaga la VPN un momento, deja que Devin Desktop instale el servidor, y vuelve a encenderla. (Ojo: cuando Devin Desktop se actualice, va a necesitar internet otra vez.)

---

## Paso 4: Borrar la instalación vieja y empezar limpio

Si hay internet pero sigue fallando, seguramente quedó basura de un intento anterior. Cierra Devin Desktop y en PowerShell ejecuta:

```powershell
wsl --shutdown
wsl -d Ubuntu -- bash -c "rm -rf ~/.devin-server ~/.windsurf-server"
```

> Esto solo borra el "servidor" de Devin dentro de Ubuntu, **no toca tus proyectos ni tus archivos**. Devin Desktop lo vuelve a instalar solito.

Si solo quieres quitar el "candado" sin borrar todo:

```powershell
wsl -d Ubuntu -- bash -c "rm -f ~/.devin-server/.installation_lock"
```

---

## Paso 5: Conectarte a WSL desde Devin Desktop

1. Abre Devin Desktop.
2. Pulsa `Ctrl + Shift + P` y escribe **"WSL: Connect to WSL"** (o haz clic en el botón de conexión remota, abajo a la izquierda).
3. Elige **Ubuntu**.
4. Espera a que termine "Setting up WSL Distro: Ubuntu" (la primera vez tarda un poquito porque descarga el servidor).
5. Ve a **Open Folder** y elige tu carpeta de Linux, por ejemplo `/home/tu_usuario/mi-proyecto`.

¡Y ya estás trabajando dentro de WSL!

---

## Paso 6: Comprobar que quedó funcionando

Si quieres asegurarte de que el servidor está instalado y corriendo:

```powershell
wsl -d Ubuntu -- bash -lc "ls ~/.devin-server/bin/ ; ps -eo pid,cmd | grep devin-server | grep -v grep"
```

- Si ves una carpeta con un nombre largo (tipo `053434014dbd...`) dentro de `bin/` y algunos procesos `devin-server` / `windsurf-cli`: está todo bien.
- Si no sale nada: vuelve al **Paso 4** y conéctate de nuevo.

---

## Paso 7: Si el error vuelve justo después de que Devin Desktop se actualice

Esto me pasó a mí: todo funcionaba, Devin Desktop se actualizó solo, lo volví a abrir y salió **otra vez** el mismo error (`Couldn't install vscode server ... non-zero exit status`). Los pasos 0 a 4 no lo arreglaron. Aquí va cómo saber si es este caso y cómo arreglarlo.

### 7.1 ¿Por qué pasa?

Cada versión de Devin Desktop necesita **su propio servidor** dentro de Ubuntu. Cuando se actualiza, intenta descargar el servidor nuevo en `~/.devin-server/bin/<código-de-versión>/`.

El problema es que, en algunas versiones, Devin Desktop **no le dice a la extensión de WSL desde dónde descargarlo** (en su archivo `product.json` falta el dato `serverDownloadUrlTemplate`). Entonces la extensión usa una dirección de reserva que es de **VSCodium** (otro editor), donde obviamente no existe el servidor de Devin. Resultado: la descarga da error 404 y queda un archivo **vacío** de 0 bytes. Por eso falla siempre, aunque tu internet funcione perfecto.

### 7.2 Comprobar si es tu caso

Abre una terminal de **Ubuntu** y escribe:

```bash
ls -la ~/.devin-server/bin/*/
```

Es este caso si ves algo así: una carpeta con un nombre largo y dentro **solo** un archivo `vscode-server.tar.gz` de tamaño **0**:

```
-rw-r--r-- 1 alvaro alvaro    0 Oct 30 05:06 vscode-server.tar.gz
```

Si quieres confirmarlo del todo, mira qué dirección usa la extensión de WSL (cambia `TU_USUARIO` por tu usuario de Windows):

```bash
grep -ohE 'https://[^"'"'"' `]*reh[^"'"'"' `]*' /mnt/c/Users/TU_USUARIO/.devin/extensions/jeanp413.open-remote-wsl-*/out/extension.js | sort -u
```

Si sale una dirección de `github.com/VSCodium/...`, es exactamente este problema.

### 7.3 Arreglarlo: instalar el servidor a mano

La idea es sencilla: descargamos nosotros el servidor correcto desde los servidores de Devin y lo dejamos en la carpeta donde la extensión lo busca. Cuando lo encuentra ya instalado, no intenta descargarlo y conecta.

1. **Cierra Devin Desktop.**

2. **Actualiza Devin Desktop a la última versión** si no lo está (el arreglo descarga el servidor de la versión más nueva, así que tu editor tiene que ser esa misma versión).

3. Abre una terminal de **Ubuntu** y pega todo este bloque:

   ```bash
   # 1) Código de versión que tu editor intentó instalar (la carpeta que quedó a medias)
   C=$(ls -t ~/.devin-server/bin | head -1); echo "Tu editor: $C"

   # 2) Preguntar a Devin cuál es el servidor más nuevo y dónde está
   INFO=$(curl -fsSL https://windsurf-stable.codeium.com/api/update/linux-reh-x64/stable/latest)
   URL=$(echo "$INFO" | grep -oE '"url":"[^"]*"' | cut -d'"' -f4)
   SHA=$(echo "$INFO" | grep -oE '"sha256hash":"[^"]*"' | cut -d'"' -f4)
   echo "Servidor: $URL"

   # 3) Comprobar que es la misma versión que tu editor
   case "$URL" in *"/$C/"*) echo "OK: coinciden";; *) echo "NO COINCIDEN: actualiza Devin Desktop y vuelve a empezar";; esac
   ```

   Si dice **`OK: coinciden`**, sigue. Si dice **`NO COINCIDEN`**, actualiza Devin Desktop (o reinstálalo) y repite este paso.

4. En la **misma** terminal, pega esto (tarda un par de minutos, son unos 200 MB):

   ```bash
   cd ~/.devin-server/bin/$C && rm -f vscode-server.tar.gz
   curl -fL -o vscode-server.tar.gz "$URL"
   echo "$SHA  vscode-server.tar.gz" | sha256sum -c - && tar -xzf vscode-server.tar.gz --strip-components 1 && ls -la bin/devin-server && rm -f vscode-server.tar.gz
   cd ~
   ```

   Tiene que salir `vscode-server.tar.gz: OK` y una línea con `bin/devin-server`. El `OK` significa que el archivo se descargó completo y sin modificar.

5. Abre Devin Desktop y conéctate a WSL (Paso 5) o pulsa **Retry**.

> **Ojo:** esto hay que repetirlo **cada vez que Devin Desktop se actualice** mientras no corrijan el `product.json`. Son solo los pasos 3 y 4 otra vez.

> **¿Y si cambio la dirección en la configuración?** La extensión tiene un ajuste `remote.WSL.serverDownloadUrlTemplate`, pero no sirve aquí: el nombre del archivo de Devin usa un número de versión distinto (por ejemplo `3.11.234`) del que la extensión conoce (`1.126.0`), así que tendrías que escribir la dirección completa a mano y cambiarla en cada actualización. La instalación manual es igual de rápida y más segura porque comprueba el checksum.

---

## Resumen rápido (chuleta)

| Problema | Qué hacer |
|---|---|
| Sale el error una vez | Pulsa **Retry** |
| WSL viejo | `wsl --update` |
| WSL sin internet (VPN) | Crea `.wslconfig` con `networkingMode=mirrored` + `wsl --shutdown` |
| Instalación a medias | `wsl -d Ubuntu -- bash -c "rm -rf ~/.devin-server ~/.windsurf-server"` |
| Falla justo después de actualizar Devin Desktop (`vscode-server.tar.gz` de 0 bytes) | Instala el servidor a mano (**Paso 7**) |
| Comprobar que funciona | `ls ~/.devin-server/bin/` dentro de WSL |

---

## Lo que pasó en mi caso (diagnóstico real)

- `wsl --version` → WSL **3.0.3**, distro Ubuntu en **WSL 2**. Todo al día.
- No tenía archivo `.wslconfig` (red NAT por defecto).
- La prueba de internet desde WSL respondió bien (`404` del servidor de descargas de Devin, `301` de Google) → **la red no era el problema**.
- Dentro de `~/.devin-server` ya estaba el servidor instalado y los procesos corriendo → un intento posterior había funcionado.
- Conclusión: **fue un fallo puntual** y con reintentar se arregló. Si hubiera seguido, el siguiente paso era el **Paso 4** (borrar e instalar limpio).

---

## Lo que pasó en mi caso, segunda parte (después de una actualización)

Unos días después Devin Desktop se actualizó a la versión **1.126.0** y al abrirlo volvió el error. Esto es lo que hice, en orden:

1. Miré qué había en `~/.devin-server`: solo una carpeta nueva `bin/92c83011.../` con un `vscode-server.tar.gz` de **0 bytes**. La descarga había fallado.
2. Probé la red desde WSL: respondía bien, así que no era internet.
3. Busqué el `product.json` de Devin Desktop (en mi caso estaba instalado en `C:\devindesktop`) y vi que **no tenía** `serverDownloadUrlTemplate`.
4. Busqué la dirección dentro de la extensión de WSL (`jeanp413.open-remote-wsl`) y salió la de **VSCodium**: por eso daba 404.
5. Le pregunté al servidor de actualizaciones de Devin (`https://windsurf-stable.codeium.com/api/update/linux-reh-x64/stable/latest`) y me dio la dirección correcta del servidor 1.126.0 y su checksum.
6. Lo descargué a mano en `~/.devin-server/bin/92c83011.../`, comprobé el checksum, lo descomprimí y volví a conectar (**Paso 7**).

---

## Fuente

Documentación oficial de Devin Desktop sobre problemas con WSL:
https://docs.devin.ai/desktop/troubleshooting/windsurf-wsl-performance
