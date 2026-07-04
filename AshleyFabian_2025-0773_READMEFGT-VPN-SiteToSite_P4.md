# Práctica P4 — VPN Site-to-Site con FortiGate

**Estudiante:** Ashley Fabian
**Matrícula:** 2025-0773
**Institución:** Instituto Tecnológico Las Américas (ITLA)
**Entorno de simulación:** GNS3 (FortiGate-VM64-KVM 7.0.9, Cisco CSR1000v 16.8.1a, Kali Linux)
**Direccionamiento base:** 25.7.73.0/24

---

## Índice

1. Introducción y objetivo general
2. VPN Site-to-Site
   2.1 Objetivo
   2.2 Topología
   2.3 Direccionamiento IP
   2.4 Parámetros de configuración
   2.5 Configuración — FGT-1
   2.6 Configuración — FGT-2
   2.7 Configuración — Router ISP (CSR1000v)
   2.8 Verificación y pruebas de conectividad
   2.9 Problemas encontrados y solución (troubleshooting)
3. Conclusión

---

## 1. Introducción y objetivo general

Esta práctica tiene como finalidad demostrar la configuración e implementación de dos modalidades de VPN IPsec utilizando appliances FortiGate: una **VPN Site-to-Site** entre dos sedes (LAN-A y LAN-B) a través de un enlace que simula Internet, y una **VPN Client-to-Site** que permite a un cliente remoto acceder de forma segura a una red interna. Ambos escenarios se implementaron en un entorno virtualizado GNS3, utilizando el direccionamiento IP 25.7.73.0/24, derivado de la matrícula del estudiante (2025-0773).

---

## 2. VPN Site-to-Site

### 2.1 Objetivo

Establecer un túnel VPN IPsec Site-to-Site entre dos FortiGates (FGT-1 y FGT-2), permitiendo la comunicación cifrada y transparente entre dos redes LAN geográficamente separadas (simuladas mediante un router intermedio que actúa como proveedor de Internet), sin exponer el tráfico interno en texto plano a través de la red pública.

### 2.2 Topología

```
[VPCS-A] --- [Switch] --- [FGT-1: port1]   [FGT-1: port2] --- [CSR1000v: Gi1]
                                                                      |
                                                              (Router ISP)
                                                                      |
                                            [FGT-2: port2] --- [CSR1000v: Gi2]
[VPCS-B] --- [Switch] --- [FGT-2: port1]
```

[INSERTAR CAPTURA: vista completa de la topología en el lienzo de GNS3, con el rótulo de nombre y matrícula visible]

![](./Topologia.png)

*Explicación:* la topología simula dos sedes independientes (LAN-A y LAN-B) conectadas a Internet a través de sus respectivos FortiGates, los cuales negocian un túnel IPsec sobre el enlace WAN para permitir comunicación segura entre ambas redes internas.

### 2.3 Direccionamiento IP

| Segmento | Red | Rango útil | Gateway |
|---|---|---|---|
| LAN-A (detrás de FGT-1) | 25.7.73.0/26 | .1 – .62 | 25.7.73.1 |
| LAN-B (detrás de FGT-2) | 25.7.73.64/26 | .65 – .126 | 25.7.73.65 |
| WAN FGT-1 ↔ Router ISP | 25.7.73.128/30 | .129 – .130 | ISP: .129 / FGT-1: .130 |
| WAN FGT-2 ↔ Router ISP | 25.7.73.132/30 | .133 – .134 | ISP: .133 / FGT-2: .134 |

| Interfaz | Equipo | IP asignada | Función |
|---|---|---|---|
| port1 | FGT-1 | 25.7.73.1/26 | Gateway de LAN-A |
| port2 | FGT-1 | 25.7.73.130/30 | Enlace WAN hacia ISP |
| port1 | FGT-2 | 25.7.73.65/26 | Gateway de LAN-B |
| port2 | FGT-2 | 25.7.73.134/30 | Enlace WAN hacia ISP |
| Gi1 | CSR1000v | 25.7.73.129/30 | Enlace hacia FGT-1 |
| Gi2 | CSR1000v | 25.7.73.133/30 | Enlace hacia FGT-2 |

### 2.4 Parámetros de configuración

| Parámetro | Valor |
|---|---|
| Tipo de VPN | IPsec Site-to-Site (modo túnel) |
| Versión IKE | IKEv2 |
| Método de autenticación | Pre-shared Key (PSK) |
| NAT entre sedes | Deshabilitado (No NAT between sites) |
| Nombre túnel FGT-1 | VPN-STS-FGT1-FGT2 |
| Nombre túnel FGT-2 | VPN-STS-FGT2-FGT1 |
| Subred local FGT-1 | 25.7.73.0/26 |
| Subred remota FGT-1 | 25.7.73.64/26 |
| Subred local FGT-2 | 25.7.73.64/26 |
| Subred remota FGT-2 | 25.7.73.0/26 |
| Acceso a Internet vía túnel | No (None) |

### 2.5 Configuración — FGT-1

**Interfaces y ruta estática** (ver archivo `AshleyFabian_2025-0773_FGT1-STS_P4.txt` para el detalle completo):

```
config system interface
    edit "port1"
        set mode static
        set ip 25.7.73.1 255.255.255.192
        set allowaccess ping https ssh http
    next
    edit "port2"
        set ip 25.7.73.130 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 25.7.73.129
        set device "port2"
    next
end
```

[INSERTAR CAPTURA: salida de `get system interface physical port1` y `port2` en FGT-1]

![](./CFGT11.png)

*Explicación:* se configuran ambas interfaces en modo estático. `port1` actúa como gateway de LAN-A con acceso administrativo habilitado (ping, https, ssh); `port2` es el enlace hacia el router ISP, con acceso limitado únicamente a ping por seguridad.

[INSERTAR CAPTURA: pantalla de Address Objects (LAN-A y LAN-B) en Policy & Objects]
![](./PoliticaKali1.png)

*Explicación:* se crean los objetos de dirección que representan ambas subredes, utilizados posteriormente por el asistente de VPN para definir el tráfico permitido a través del túnel.

[INSERTAR CAPTURA: configuración de Phase 1 y Phase 2 del túnel VPN-STS-FGT1-FGT2 (VPN > IPsec Tunnels > Edit)]
![](./EditVPNTunnel.png)

*Explicación:* el túnel se configuró mediante el asistente de VPN IPsec de FortiGate, seleccionando el template "Site to Site" con configuración personalizada. Se utilizó IKEv2 con autenticación por llave pre-compartida, apuntando a la IP WAN remota 25.7.73.134.

[INSERTAR CAPTURA: políticas de firewall generadas automáticamente por el asistente]

![](./PoliticaKali1.png)

*Explicación:* el asistente generó automáticamente dos políticas de firewall (una por cada sentido del tráfico) que permiten la comunicación entre LAN-A y LAN-B exclusivamente a través de la interfaz de túnel.

### 2.6 Configuración — FGT-2

**Interfaces y ruta estática** (ver archivo `AshleyFabian_2025-0773_FGT2-STS_P4.txt`):

```
config system interface
    edit "port1"
        set mode static
        set ip 25.7.73.65 255.255.255.192
        set allowaccess ping https ssh http
    next
    edit "port2"
        set ip 25.7.73.134 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 25.7.73.133
        set device "port2"
    next
end
```

[INSERTAR CAPTURA: salida de `get system interface physical port1` y `port2` en FGT-2]

![](./CFGT21.png)

[INSERTAR CAPTURA: configuración de Phase 1 y Phase 2 del túnel VPN-STS-FGT2-FGT1]
![](./EditVPNTunnel.png)

*Explicación:* configuración simétrica a FGT-1, con la IP remota apuntando a 25.7.73.130 (WAN de FGT-1) y la misma llave pre-compartida, requisito indispensable para que ambos extremos logren negociar la fase de autenticación IKE correctamente.

### 2.7 Configuración — Router ISP (CSR1000v)

(ver archivo `AshleyFabian_2025-0773_Router-ISP_P4.txt`):

```
interface GigabitEthernet1
 ip address 25.7.73.129 255.255.255.252
 no shutdown
interface GigabitEthernet2
 ip address 25.7.73.133 255.255.255.252
 no shutdown
```

[INSERTAR CAPTURA: `show ip interface brief` en el CSR1000v]
![](./CISP.png)

*Explicación:* el router actúa únicamente como intermediario que simula el proveedor de Internet; no tiene visibilidad de las redes LAN internas, ya que ese tráfico viaja encapsulado dentro del túnel IPsec entre los dos FortiGates.

### 2.8 Verificación y pruebas de conectividad

[INSERTAR CAPTURA: VPN > IPsec Tunnels mostrando el túnel en estado UP (verde) en ambos FortiGates]

![](./TunnelUPKali1.png)
![](./TunnelUPKali2.png)

[INSERTAR CAPTURA: consola de VPCS-A ejecutando `ping 25.7.73.66` y `trace 25.7.73.66`]
![](./PC1.png)

[INSERTAR CAPTURA: consola de VPCS-B ejecutando `ping 25.7.73.2` y `trace 25.7.73.66`]
![](./PC2.png)

*Explicación:* el resultado del trace-route confirma el correcto funcionamiento del túnel: desde PC1 (VPCS-A), el segundo salto corresponde directamente a la IP interna de FGT-2 (25.7.73.65), **sin mostrar los saltos intermedios del router ISP** (25.7.73.129 / 25.7.73.133). Esto demuestra que el tráfico entre ambas LANs viaja completamente encapsulado y cifrado dentro del túnel IPsec, y no por la ruta física normal a través del proveedor de Internet simulado.

### 2.9 Problemas encontrados y solución (troubleshooting)

Durante la configuración se presentaron dos incidencias relevantes, documentadas aquí por su valor técnico:

**a) Interfaz `port1` en modo DHCP no aplicaba la IP estática.** Al configurar `port1` únicamente con `set ip`, la interfaz permanecía en modo DHCP (valor por defecto de esta imagen) con IP `0.0.0.0`, impidiendo cualquier conectividad. Se identificó mediante el comando `get system interface physical port1`, y se solucionó agregando explícitamente `set mode static` antes de asignar la IP.

**b) Acceso HTTPS a la GUI de administración fallaba con "Unable to connect".** Se diagnosticó mediante `diagnose sniffer packet` que el handshake TCP se completaba correctamente, pero el establecimiento de la sesión TLS era reiniciado (`RST`) sistemáticamente tras el envío del *ClientHello*. Tras descartar causas de hardware (entropía del generador de números aleatorios, driver de red virtio, modelo de CPU virtual), se determinó que la causa raíz era el modo **HTTPS-Only** de Firefox, que forzaba la reescritura de cualquier URL a HTTPS antes de intentar la conexión, chocando con una incompatibilidad puntual del certificado autofirmado del FortiGate en este entorno de virtualización anidada. La solución fue deshabilitar HTTPS-Only Mode en el navegador y acceder por HTTP en el entorno de laboratorio (práctica no recomendada en producción).

---

## 3. Conclusión

Se logró implementar y verificar exitosamente una VPN Site-to-Site tipo IPsec entre dos FortiGates (FGT-1 y FGT-2), utilizando IKEv2 con autenticación mediante llave pre-compartida, sobre un enlace WAN simulado a través de un router intermedio (CSR1000v) que representa un proveedor de Internet. La configuración de interfaces, rutas estáticas, objetos de dirección y políticas de firewall en ambos extremos permitió establecer un túnel cifrado y funcional, confirmado mediante el estado "UP" visible en la GUI de ambos dispositivos.
Las pruebas de conectividad realizadas desde VPCS-A y VPCS-B validaron el correcto funcionamiento del túnel: los resultados de trace-route mostraron que el tráfico entre ambas LANs viaja directamente entre las interfaces internas de los FortiGates, sin exponer ni atravesar visiblemente los saltos del router ISP, lo cual demuestra que la comunicación ocurre completamente encapsulada y cifrada dentro del túnel IPsec, tal como se espera de una VPN Site-to-Site correctamente configurada.
Durante el desarrollo de la práctica se documentaron dos incidencias técnicas de valor formativo: la necesidad de establecer explícitamente el modo estático en las interfaces WAN/LAN antes de asignar una IP (dado que la imagen del FortiGate inicia sus interfaces en modo DHCP por defecto), y un conflicto de acceso HTTPS al GUI administrativo causado por el modo "HTTPS-Only" del navegador Firefox en combinación con el certificado autofirmado del FortiGate en un entorno de virtualización anidada. Ambos problemas fueron diagnosticados de forma metódica —descartando primero causas de hardware y capa de transporte antes de aislar la causa real— y resueltos sin comprometer la integridad de la configuración de red.
En conjunto, esta práctica reforzó el entendimiento de los mecanismos de negociación IKE, el rol de las políticas de firewall en el reenvío de tráfico cifrado, y la importancia de un direccionamiento IP planificado (derivado del bloque institucional 25.7.73.0/24) para mantener consistencia y trazabilidad a lo largo de las distintas topologías VPN evaluadas en el curso.
