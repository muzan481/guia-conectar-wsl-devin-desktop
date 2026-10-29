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

## Resumen rápido (chuleta)

| Problema | Qué hacer |
|---|---|
| Sale el error una vez | Pulsa **Retry** |
| WSL viejo | `wsl --update` |
| WSL sin internet (VPN) | Crea `.wslconfig` con `networkingMode=mirrored` + `wsl --shutdown` |
| Instalación a medias | `wsl -d Ubuntu -- bash -c "rm -rf ~/.devin-server ~/.windsurf-server"` |
| Comprobar que funciona | `ls ~/.devin-server/bin/` dentro de WSL |

---

## Lo que pasó en mi caso (diagnóstico real)

- `wsl --version` → WSL **3.0.3**, distro Ubuntu en **WSL 2**. Todo al día.
- No tenía archivo `.wslconfig` (red NAT por defecto).
- La prueba de internet desde WSL respondió bien (`404` del servidor de descargas de Devin, `301` de Google) → **la red no era el problema**.
- Dentro de `~/.devin-server` ya estaba el servidor instalado y los procesos corriendo → un intento posterior había funcionado.
- Conclusión: **fue un fallo puntual** y con reintentar se arregló. Si hubiera seguido, el siguiente paso era el **Paso 4** (borrar e instalar limpio).

---

## Fuente

Documentación oficial de Devin Desktop sobre problemas con WSL:
https://docs.devin.ai/desktop/troubleshooting/windsurf-wsl-performance
