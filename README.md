# Práctica 2 - Infraestructura 3: Acceso Remoto Híbrido (Web Público vía VIP + SSH Estrictamente por VPN)

* **Estudiante:** Cristopher Navarro  
* **Matrícula:** 2025-0720  
* **Asignatura:** Seguridad de Redes  
* **Docente:** Jonathan Esteban Rondón Corniel  
* **Institución:** Instituto Tecnológico de Las Américas (ITLA)  
* **Repositorio GitHub:** https://github.com/CristopherNavarro/Tarea3_SeguridadRedes_P3_CristopherNavarro_20250720  
* **Fecha de Realización:** Octubre 2026  

---

## 1. Enlace al Video Demostrativo en YouTube
* **URL del Video:** https://youtu.be/HZo9L9plypU  
*(Video explicativo con cámara y micrófono activos, mostrando en pantalla fecha/hora, topología GNS3, CLI de Cisco, Web GUI de FortiOS y pruebas de consola en vivo).*

> **Nota aclaratoria del autor sobre la duración del video:**  
> El video tiene una duración de aproximadamente 10 minutos y 30 segundos, superando por escasos 30 segundos el límite establecido. Ofrezco mis más sinceras disculpas al profesor por esta ligera extensión de tiempo; la misma fue el resultado de procurar explicar detallada y pausadamente cada fundamento técnico, la verificación exhaustiva de la regla Virtual IP (VIP), el túnel IPsec en CLI/GUI y la comprobación en tiempo real del bloqueo del puerto SSH al suspender la política de seguridad, asegurando una demostración rigurosa e integral sin omitir ningún criterio de evaluación.

---

## 2. Propósito y Objetivos del Laboratorio

El propósito fundamental de esta práctica consistió en diseñar, implementar, auditar y certificar una arquitectura de **Acceso Remoto Híbrido y Segregación Perimetral de Servicios**, donde coexisten dos requerimientos de seguridad divergentes:
1. **Publicación de Servicios Web Públicos:** Permitir el acceso general desde Internet hacia un servicio web seguro (HTTPS puerto 443) alojado en un servidor interno, utilizando una **Virtual IP (VIP)** con Port Forwarding / DNAT en el firewall perimetral FortiGate, **sin requerir túnel VPN**.
2. **Acceso Administrativo Altamente Restringido:** Prohibir de forma terminante cualquier intento de acceso administrativo hacia el puerto SSH (puerto 22) desde la red pública (WAN), obligando a que toda gestión remota transite de forma cifrada y autenticada a través de un **túnel VPN IPsec Site-to-Site**.

### Objetivos Específicos Alcanzados:
1. Diseñar e implementar el direccionamiento IP personalizado derivado estrictamente de mi número de matrícula institucional (`2025-0720`).
2. Configurar en FortiGate un objeto Virtual IP (`VIP_WEB_SERVER`) que mapea la dirección pública `203.25.7.10:443` hacia la IP interna del servidor `10.25.7.130:443`.
3. Establecer políticas de firewall perimetrales diferenciadas:
   * Autorizar exclusivamente tráfico `HTTPS` desde la WAN hacia el objeto VIP.
   * Autorizar tráfico `SSH` (puerto 22) e `ICMP` exclusivamente desde la interfaz virtual del túnel IPsec hacia el segmento interno de servidores.
4. Desplegar el túnel VPN IPsec Site-to-Site entre el router Cisco 7200 y el FortiGate, asegurando la convergencia de IKEv1 Fase 1 (`QM_IDLE`) y Fase 2 con Diffie-Hellman Grupo 14 (PFS).
5. Desplegar en el servidor de destino tanto el demonio HTTPS en el puerto 443 como el servidor OpenSSH (`sshd`) en el puerto 22.
6. Demostrar la independencia y enforzamiento de políticas: al suspender la política VPN, el tráfico administrativo se bloquea al 100 %, mientras que el servicio web público por VIP permanece ininterrumpido.

---

## 3. Topología de Red y Diagrama de Conectividad

La topología fue construida y cableada en GNS3 respaldado por VMware Workstation:

```text
[PC-Usuario] (10.25.7.10/25 - VLAN 10)
      | (eth0)
      | (e1 - Access VLAN 10)
[SW-SiteA]
      | (e0 - 802.1Q Trunk)
      | (f0/1.10)
[R-Cisco] (Sitio A - Cisco 7200 IOS 15.2)
      | (f0/0: 203.25.7.2/29)
      |
      | (f0/0: 203.25.7.1/29)
    [ISP]
      | (f0/1: 203.25.7.9/29)
      |
      | (port1: 203.25.7.10/29 - WAN Pública / VIP)
[FortiGate] (Sitio B - FortiOS 7.0.9)
      | (port2: 10.25.7.129/28)
      | (eth0)
[Servidor-Web] (10.25.7.130/28 - HTTPS:443 / SSH:22)
```

---

## 4. Esquema de Direccionamiento IP (Matrícula: 2025-0720)

Aplicando los parámetros exigidos en la guía académica con base en mi matrícula:

| Dispositivo | Interfaz Física / Lógica | Rol / Segmento | Dirección IP | Máscara de Red | Puerta de Enlace |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PC-Usuario** | `eth0` | LAN Usuarios (VLAN 10) | `10.25.7.10` (DHCP) | `255.255.255.128` (/25) | `10.25.7.1` |
| **SW-SiteA** | `e0` (Trunk) / `e1` (Acceso) | Conmutación Local | N/A | N/A | N/A |
| **R-Cisco** | `FastEthernet0/1.10` | Gateway LAN Sitio A (VLAN 10) | `10.25.7.1` | `255.255.255.128` (/25) | N/A |
| **R-Cisco** | `FastEthernet0/0` | WAN Sitio A | `203.25.7.2` | `255.255.255.248` (/29) | `203.25.7.1` |
| **ISP** | `FastEthernet0/0` | Enlace WAN Sitio A | `203.25.7.1` | `255.255.255.248` (/29) | N/A |
| **ISP** | `FastEthernet0/1` | Enlace WAN Sitio B | `203.25.7.9` | `255.255.255.248` (/29) | N/A |
| **ISP** | `Loopback0` | Simulación DNS/Internet | `8.8.8.8` | `255.255.255.255` (/32) | N/A |
| **FortiGate** | `port1` | WAN Pública (Sede VIP) | `203.25.7.10` | `255.255.255.248` (/29) | `203.25.7.9` |
| **FortiGate** | `port2` | Gateway LAN Sitio B | `10.25.7.129` | `255.255.255.240` (/28) | N/A |
| **FortiGate** | `port3` | Gestión Web GUI | `192.168.6.202` | `255.255.255.0` (/24) | N/A |
| **Servidor-Web** | `eth0` | Servidor Web y SSH | `10.25.7.130` | `255.255.255.240` (/28) | `10.25.7.129` |

---

## 5. Procedimiento de Configuración Implementado

### 5.1. Configuración de FortiGate (Publicación VIP, Túnel IPsec y Políticas)

1. **Definición del Objeto Virtual IP (VIP):**
```fortios
config firewall vip
    edit "VIP_WEB_SERVER"
        set extip 203.25.7.10
        set mappedip "10.25.7.130"
        set extintf "port1"
        set portforward enable
        set protocol tcp
        set extport 443
        set mappedport 443
    next
end
```

2. **Configuración del Túnel IPsec Site-to-Site:**
```fortios
config vpn ipsec phase1-interface
    edit "VPN-S2S-FGT"
        set interface "port1"
        set peertype any
        set net-device disable
        set dpd on-idle
        set dhgrp 14
        set proposal des-sha256
        set remote-gw 203.25.7.2
        set psksecret ClaveSegura2025
    next
end

config vpn ipsec phase2-interface
    edit "VPN-S2S-FGT"
        set phase1name "VPN-S2S-FGT"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 10.25.7.128 255.255.255.240
        set dst-subnet 10.25.7.0 255.255.255.128
        set auto-negotiate enable
    next
end

config router static
    edit 2
        set dst 10.25.7.0 255.255.255.128
        set device "VPN-S2S-FGT"
    next
end
```

3. **Políticas de Firewall Segregadas:**
```fortios
config firewall address
    edit "LAN_SITE_B"
        set subnet 10.25.7.128 255.255.255.240
    next
    edit "LAN_SITE_A"
        set subnet 10.25.7.0 255.255.255.128
    next
end

config firewall policy
    # 1. Acceso Web Público hacia el VIP (Sin VPN)
    edit 10
        set name "WAN_to_VIP_HTTPS"
        set srcintf "port1"
        set dstintf "port2"
        set action accept
        set srcaddr "all"
        set dstaddr "VIP_WEB_SERVER"
        set schedule "always"
        set service "HTTPS"
    next
    # 2. Acceso Administrativo Restringido por VPN (SSH + PING)
    edit 20
        set name "VPN_to_LAN_SSH_PING"
        set srcintf "VPN-S2S-FGT"
        set dstintf "port2"
        set action accept
        set srcaddr "LAN_SITE_A"
        set dstaddr "LAN_SITE_B"
        set schedule "always"
        set service "SSH" "PING"
    next
    # 3. Retorno LAN hacia VPN
    edit 21
        set name "LAN_to_VPN"
        set srcintf "port2"
        set dstintf "VPN-S2S-FGT"
        set action accept
        set srcaddr "LAN_SITE_B"
        set dstaddr "LAN_SITE_A"
        set schedule "always"
        set service "ALL"
    next
end
```

### 5.2. Configuración de R-Cisco (Sitio A - Cisco IOS 15.2)

1. **Subinterfaz LAN VLAN 10 y Servidor DHCP:**
```cisco
hostname R-Cisco
no ip domain-lookup

interface FastEthernet0/1
 no shutdown

interface FastEthernet0/1.10
 encapsulation dot1Q 10
 ip address 10.25.7.1 255.255.255.128
 ip nat inside
 no shutdown

ip dhcp excluded-address 10.25.7.1 10.25.7.9
ip dhcp excluded-address 10.25.7.51 10.25.7.126
ip dhcp pool POOL_VLAN10
 network 10.25.7.0 255.255.255.128
 default-router 10.25.7.1
 dns-server 8.8.8.8
```

2. **WAN, NAT y Reglas IPsec:**
```cisco
interface FastEthernet0/0
 description Enlace WAN hacia ISP
 ip address 203.25.7.2 255.255.255.248
 ip nat outside
 no shutdown

ip route 0.0.0.0 0.0.0.0 203.25.7.1

crypto isakmp policy 10
 encr des
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ClaveSegura2025 address 203.25.7.10

crypto ipsec transform-set TS-VPN esp-des esp-sha256-hmac
 mode tunnel

ip access-list extended ACL-VPN
 permit ip 10.25.7.0 0.0.0.127 10.25.7.128 0.0.0.15

# NAT Exemption para tráfico VPN y NAT Overload para tráfico hacia Internet/VIP
ip access-list extended ACL-NAT
 deny ip 10.25.7.0 0.0.0.127 10.25.7.128 0.0.0.15
 permit ip 10.25.7.0 0.0.0.127 any

ip nat inside source list ACL-NAT interface FastEthernet0/0 overload

crypto map VPN-MAP 10 ipsec-isakmp
 set peer 203.25.7.10
 set transform-set TS-VPN
 set pfs group14
 match address ACL-VPN

interface FastEthernet0/0
 crypto map VPN-MAP
```

---

## 6. Evidencias de Verificación y Auditoría en Vivo

### 6.1. Prueba A: Acceso Web Público mediante VIP (Directo sin VPN)
```text
/ # curl -k -i https://203.25.7.10/
HTTP/1.1 200 OK
Server: SimpleHTTP/0.6 Python/3.14.8
Date: Fri, 02 Oct 2026 00:45:05 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 1006
Connection: close

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Servidor Web Seguro - Practica 2</title>
</head>
<body style="font-family: Arial, sans-serif; background-color: #f4f6f9; color: #333; padding: 30px;">
    <div style="background: white; border: 1px solid #ddd; padding: 25px; border-radius: 8px; max-width: 700px; margin: 0 auto; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
        <h1 style="color: #0b5394;">Servidor Web Seguro (HTTPS)</h1>
        <hr style="border: 0; border-top: 1px solid #ccc;">
        <p><strong>Estudiante:</strong> Cristopher Navarro</p>
        <p><strong>Matricula:</strong> 2025-0720</p>
        <p><strong>Asignatura:</strong> Seguridad de Redes</p>
        <p><strong>Docente:</strong> Jonathan Esteban Rondon Corniel</p>
        <p><strong>Segmento Servidor:</strong> 10.25.7.128/28 (IP: 10.25.7.130)</p>
        <p style="color: #274e13; font-weight: bold;">Acceso exitoso a traves del tunel VPN IPsec Site-to-Site!</p>
    </div>
</body>
</html>
```
*Interpretación técnica:* La estación `PC-Usuario` apunta a la IP pública `203.25.7.10`. El firewall traduce la petición mediante el objeto VIP y la entrega al servidor web en la `10.25.7.130:443`, respondiendo con HTTP 200 OK sin requerir túnel VPN.

### 6.2. Prueba B: Bloqueo de SSH desde la Red WAN
```text
/ # nc -z -v -w 3 203.25.7.10 22
nc: 203.25.7.10 (203.25.7.10:22): Operation timed out
```
*Interpretación técnica:* Al intentar acceder al puerto SSH (22) en la IP pública `203.25.7.10`, la conexión expira. El firewall FortiGate descarta el tráfico en virtud de su política de denegación implícita, demostrando que la administración no está expuesta a Internet.

### 6.3. Prueba C: Acceso Administrativo Seguro por SSH (Vía Túnel VPN)
```text
/ # ssh -o StrictHostKeyChecking=no root@10.25.7.130 "hostname && uname -a"
Servidor-Web
Linux Servidor-Web 7.0.0-34-generic #34-Ubuntu SMP PREEMPT_DYNAMIC Wed Sep  2 14:29:37 UTC 2026 x86_64 Linux
```
*Interpretación técnica:* Al conectar por SSH hacia la dirección IP privada `10.25.7.130`, el tráfico coincide con la lista `ACL-VPN` en Cisco, es encapsulado en IPsec, atraviesa el túnel, y es aceptado por la política `VPN_to_LAN_SSH_PING` en FortiGate, permitiendo la administración segura.

### 6.4. Prueba D: Conectividad ICMP por el Túnel VPN (Ping)
```text
/ # ping -c 5 10.25.7.130
PING 10.25.7.130 (10.25.7.130): 56 data bytes
64 bytes from 10.25.7.130: seq=0 ttl=62 time=42.784 ms
64 bytes from 10.25.7.130: seq=1 ttl=62 time=29.848 ms
64 bytes from 10.25.7.130: seq=2 ttl=62 time=42.198 ms
64 bytes from 10.25.7.130: seq=3 ttl=62 time=40.031 ms
64 bytes from 10.25.7.130: seq=4 ttl=62 time=46.404 ms

--- 10.25.7.130 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 29.848/40.253/46.404 ms
```

### 6.5. Prueba E: Demostración de Segregación y Enforzamiento (Drop Test)
1. **Deshabilitación de la política `VPN_to_LAN_SSH_PING` en FortiGate:**
   * Ping al servidor privado:
     ```text
     / # ping -c 3 -W 1 10.25.7.130
     PING 10.25.7.130 (10.25.7.130): 56 data bytes

     --- 10.25.7.130 ping statistics ---
     3 packets transmitted, 0 packets received, 100% packet loss
     ```
   * Consulta al servicio Web Público vía VIP en paralelo:
     ```text
     / # curl -k -s -o /dev/null -w "%{http_code}\n" https://203.25.7.10/
     200
     ```
   * *Resultado Demostrado:* El tráfico del túnel cae al **100 % de pérdida**, mientras que el acceso web público por VIP permanece **100 % operativo (`HTTP 200 OK`)**, confirmando la total independencia y aislamiento de los flujos.

2. **Reactivación de la política `VPN_to_LAN_SSH_PING` en FortiGate:**
   * Recuperación inmediata del tráfico VPN:
     ```text
     / # ping -c 3 10.25.7.130
     3 packets transmitted, 3 packets received, 0% packet loss
     ```
   * Acceso SSH restaurado:
     ```text
     / # ssh root@10.25.7.130 "hostname"
     Servidor-Web
     ```

---

## 7. Conclusiones Técnicas

1. **Aislamiento de Planos de Gestión y Datos:** Se validó exitosamente una de las mejores prácticas de ciberseguridad: desacoplar la exposición de servicios al público (HTTPS mediante VIP) del plano de gestión administrativa (SSH), el cual debe permanecer estrictamente confinado dentro de una VPN cifrada.
2. **Control Granular por Servicios en Políticas de Firewall:** La capacidad de filtrar por servicios específicos (`HTTPS` para la WAN, `SSH` y `PING` para la VPN) garantiza que ningún puerto no autorizado quede expuesto inadvertidamente.
3. **Robustez y Flexibilidad Perimetral:** La solución integra servicios NAT Overload saliente, NAT de destino entrante (VIP) y túneles IPsec Site-to-Site sin conflictos de enrutamiento ni superposición de direcciones.
