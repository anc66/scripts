# Pruebas de conectividad para Active Directory con PortQry

Paso a paso práctico, comandos listos para copiar y lectura de resultados.

> **Objetivo**
>
> Confirmar, desde el servidor que presenta el problema, qué puertos del controlador de dominio están disponibles, cerrados o filtrados.

## Contenido

- [1. Ejecutar las pruebas](#1-ejecutar-las-pruebas)
- [2. Cómo leer el resultado](#2-cómo-leer-el-resultado)
- [3. Prompt para analizar la salida de PortQry](#3-prompt-para-analizar-la-salida-de-portqry)
- [4. Resultados](#4-resultados)

## 1. Ejecutar las pruebas

### Prueba rápida de puertos principales: TCP

**Qué hace:** consulta los puertos TCP esenciales de Active Directory.

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\All_ports.txt -p tcp -o 53,88,135,389,445,464,636,3268,3269
```

### Validar DNS, Kerberos, NTP y LDAP: UDP

**Qué hace:** confirma la conectividad UDP con los servicios indicados.

```powershell
.\portqry.exe -n 10.10.10.10 -p udp -o 53,88,123,389,464
```

### Descubrir los puertos RPC dinámicos en escucha

**Qué hace:** consulta el RPC Endpoint Mapper y guarda los puertos que usa el servidor en ese momento.

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\rpc_epm.txt -p tcp -e 135 -y
```

### Extraer los puertos RPC encontrados

**Qué hace:** obtiene una lista única de los puertos TCP indicados por el Endpoint Mapper.

```powershell
$p = Select-String -Path .\rpc_epm.txt -Pattern 'ncacn_ip_tcp:[^\[\]]*\[(\d+)\]' |
    ForEach-Object { [int]$_.Matches[0].Groups[1].Value } |
    Sort-Object -Unique
```

### Probar los puertos RPC descubiertos

**Qué hace:** verifica únicamente los puertos que el servidor acaba de publicar.

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\rpc_epm_test.txt -p tcp -o ($p -join ',')
```

## 2. Cómo leer el resultado

| Estado | Qué significa | Acción recomendada |
|---|---|---|
| **LISTENING** | El puerto responde y hay un servicio escuchando. | El puerto está disponible. |
| **NOT LISTENING** | El host respondió, pero ningún servicio escucha en ese puerto. | Validar que el rol o servicio corresponda y esté iniciado. |
| **FILTERED** | No hubo respuesta. Puede existir un bloqueo de firewall, ACL, ruta o un host no alcanzable. | Revisar las reglas de firewall, las ACL y el enrutamiento. |
| **LISTENING or FILTERED** | Resultado UDP ambiguo. No confirma éxito ni falla. | Confirmar mediante una prueba específica del servicio. |

### Validaciones especiales

- **UDP 88:** `LISTENING or FILTERED` puede ser normal si TCP 88 aparece como `LISTENING`.
- **UDP 123:** confirmar mediante:

  ```powershell
  w32tm /stripchart /computer:10.10.10.10 /samples:3 /dataonly
  ```

- **UDP 389:** si no muestra datos, confirmar mediante:

  ```powershell
  nltest /dsgetdc:contoso.local /force
  ```

- **TCP 636 o 3269 con `NOT LISTENING`:** puede faltar un certificado válido. No necesariamente indica un problema de red.
- **TCP 135 con `LISTENING` y puertos RPC con `FILTERED`:** el firewall permite el RPC Endpoint Mapper, pero bloquea el rango RPC dinámico.

### Identificar puertos TCP en escucha en el servidor de destino

**Qué hace:** con privilegios administrativos identifica los puertos TCP requeridos que se encuentran en estado `Listen`.

```powershell
Get-NetTCPConnection -State Listen |
    Where-Object LocalPort -in 53,88,135,389,445,464,636,3268,3269 |
    Sort-Object LocalPort |
    Format-Table LocalAddress, LocalPort, State, OwningProcess
```

## 3. Prompt para analizar la salida de PortQry

Copie el siguiente texto en Copilot o en su asistente de IA y pegue al final la salida completa de PortQry:

```text
Analiza la salida de PortQry de los archivos adjuntos sin asumir datos que no aparezcan en ella.

Identifica cada combinación de puerto y protocolo y entrégala en una tabla con estas columnas:
Puerto, Protocolo, Servicio o etiqueta, Estado exacto de PortQry, Clasificación
(Disponible, No escucha, Filtrado o Ambiguo), Evidencia textual, Impacto probable y
Acción recomendada.

Reglas de interpretación:
- LISTENING = Disponible.
- NOT LISTENING = El host respondió, pero el servicio no escucha.
- FILTERED = Sin respuesta; posible firewall, ACL, ruta o host no alcanzable.
- LISTENING or FILTERED en UDP = Ambiguo; no lo marques como falla automáticamente.
- Para UDP 88, 123 y 389, agrega la validación alternativa apropiada.
- Si TCP 135 está LISTENING, pero un puerto ncacn_ip_tcp publicado por el Endpoint
  Mapper está FILTERED, concluye que el acceso RPC está incompleto.
- Si 636 o 3269 están NOT LISTENING mientras 389 o 3268 están LISTENING, indica una
  posible falta de certificado, no un bloqueo de red.

Después de la tabla, incluye:
1. Un resumen ejecutivo de máximo cinco líneas que contenga el total de puertos por estado.
2. Los resultados ambiguos que necesitan otra prueba.

SALIDA DE PORTQRY en tres archivos:
- rpc_epm.txt: puertos altos de RPC en escucha.
- All_ports.txt: resultado de todos los puertos de la prueba; incluye resultados del
  archivo rpc_epm.txt.
- rpc_epm_test.txt: resultado de los puertos altos de RPC.

No arrojes resultados duplicados para los mismos puertos presentes en distintos archivos.
```

## 4. Resultados

Complete esta sección con los resultados obtenidos durante las pruebas.

| Puerto | Protocolo | Servicio | Estado | Observaciones |
|---:|:---:|---|---|---|
| | | | | |

> **Nota:** PortQry valida alcanzabilidad y escucha. No confirma permisos, autenticación, validez de certificados ni el funcionamiento completo de la aplicación.
