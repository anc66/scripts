# Pruebas de conectividad para Active Directory con PortQry

## Introducción

Durante actividades de troubleshooting de Active Directory es habitual encontrar síntomas que inicialmente parecen asociados a autenticación, políticas de grupo, replicación o resolución de nombres, pero cuyo origen real corresponde a restricciones de conectividad entre servidores y controladores de dominio.

Este procedimiento utiliza PortQry para validar la disponibilidad de los principales puertos requeridos por Active Directory y ayudar a determinar si el problema está relacionado con servicios no disponibles, reglas de firewall, listas de control de acceso (ACL) u otro tipo de problemas de conectividad.

> **Importante**
>
> PortQry permite validar alcanzabilidad y estado de escucha de puertos. No valida autenticación, permisos, funcionamiento del servicio, integridad de Active Directory ni validez de certificados.

---

## Objetivo

Desde el servidor afectado, validar qué puertos requeridos por Active Directory se encuentran:

- Disponibles y respondiendo.
- Cerrados o sin servicios asociados.
- Filtrados por dispositivos de red o firewalls.
- Publicados dinámicamente mediante RPC.

---

## Cuándo utilizar este procedimiento

Este procedimiento suele ser útil cuando se presentan escenarios como:

- Errores de autenticación Kerberos.
- Fallas de aplicación de Group Policy.
- Problemas de incorporación de equipos al dominio.
- Errores de replicación entre controladores de dominio.
- Demoras durante inicio de sesión.
- Fallas de comunicación entre aplicaciones y Active Directory.
- Sospechas de filtrado de tráfico por firewalls o segmentación de red.

---

## Contenido

1. Pruebas TCP esenciales
2. Pruebas UDP
3. Descubrimiento de puertos RPC dinámicos
4. Interpretación de resultados
5. Validaciones adicionales recomendadas
6. Prompt para análisis con IA
7. Resultados

---

# 1. Pruebas TCP esenciales

## Consideraciones previas

Antes de ejecutar las pruebas:

- Utilice la dirección IP del controlador de dominio destino.
- Ejecute PortQry desde el servidor que presenta el problema.
- Evite realizar las pruebas desde estaciones de administración o jump servers, ya que podrían existir rutas de red diferentes.

### Puertos TCP principales

La siguiente prueba valida los servicios más relevantes utilizados por un controlador de dominio:

| Puerto | Servicio |
|----------|----------|
| 53 | DNS |
| 88 | Kerberos |
| 135 | RPC Endpoint Mapper |
| 389 | LDAP |
| 445 | SMB |
| 464 | Kerberos Password Change |
| 636 | LDAPS |
| 3268 | Global Catalog |
| 3269 | Global Catalog SSL |

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\All_ports.txt -p tcp -o 53,88,135,389,445,464,636,3268,3269
```

### Observación práctica

En muchos incidentes se observa que los puertos principales responden correctamente mientras que los puertos RPC dinámicos se encuentran bloqueados. En esos casos aplicaciones como ADUC, GPMC, DNS Manager o herramientas de administración remota pueden fallar aunque LDAP y Kerberos funcionen normalmente.

---

# 2. Validar DNS, Kerberos, NTP y LDAP mediante UDP

Algunos protocolos continúan utilizando UDP para determinadas operaciones.

```powershell
.\portqry.exe -n 10.10.10.10 -p udp -o 53,88,123,389,464
```

### Puertos evaluados

| Puerto | Servicio |
|----------|----------|
| 53 | DNS |
| 88 | Kerberos |
| 123 | NTP |
| 389 | LDAP |
| 464 | Kerberos Password Change |

### Observación práctica

Los resultados UDP deben interpretarse con cautela. Es normal que algunos dispositivos de red o sistemas operativos no respondan de la misma forma que lo hacen ante conexiones TCP.

Por este motivo, un resultado `LISTENING or FILTERED` no debe considerarse automáticamente una falla de conectividad.

---

# 3. Descubrir puertos RPC dinámicos

Active Directory utiliza RPC dinámico para múltiples operaciones administrativas.

Esta prueba consulta el Endpoint Mapper para identificar qué puertos publicados está utilizando el servidor.

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\rpc_epm.txt -p tcp -e 135 -y
```

### Por qué es importante

Es frecuente encontrar firewalls configurados para permitir únicamente el puerto TCP 135.

Aunque esto permite comunicarse con el Endpoint Mapper, las operaciones posteriores pueden fallar si el firewall bloquea los puertos dinámicos publicados por dicho servicio.

---

# 4. Extraer los puertos RPC identificados

```powershell
$p = Select-String -Path .\rpc_epm.txt -Pattern 'ncacn_ip_tcp:[^\[\]]*\[(\d+)\]' |
    ForEach-Object { [int]$_.Matches[0].Groups[1].Value } |
    Sort-Object -Unique
```

### Resultado esperado

El comando genera una colección única de puertos TCP utilizados actualmente por el controlador de dominio.

---

# 5. Validar los puertos RPC publicados

```powershell
.\portqry.exe -n 10.10.10.10 -l C:\config\rpc_epm_test.txt -p tcp -o ($p -join ',')
```

### Observación práctica

Si TCP 135 responde correctamente pero varios puertos RPC dinámicos aparecen como `FILTERED`, normalmente existe un firewall intermedio bloqueando el rango RPC dinámico.

Este patrón es uno de los problemas de conectividad más frecuentes observados en entornos segmentados.

---

# 6. Interpretación de resultados

## Estados posibles

| Estado | Significado | Acción recomendada |
|----------|----------|----------|
| LISTENING | El servicio responde y existe un proceso escuchando. | Considerar el puerto disponible. |
| NOT LISTENING | El host responde pero no existe un servicio asociado. | Validar configuración y servicios. |
| FILTERED | No se recibió respuesta. | Revisar firewall, ACL 

# 7. Apoyo en AI para interpretar resultados

## Prompt para analizar la salida de PortQry

Copiar el siguiente texto en Copilot o  asistente de IA favorito:

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
---
