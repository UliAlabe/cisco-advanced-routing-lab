# 🚀 Cisco IOS: Guía de Configuración Corporativa Multi-Sitio

Este documento centraliza los comandos de configuración para el despliegue de una red corporativa jerárquica. Abarca desde la conmutación de área local hasta el enrutamiento dinámico, servicios y políticas de seguridad perimetral.

---

## 📑 Índice Rápido
1. [Fundamentos del Sistema](#1-fundamentos-del-sistema)
2. [Capa 2: LAN y Conmutación](#2-capa-2-lan-y-conmutación)
3. [Capa 3: Enrutamiento Local y WAN](#3-capa-3-enrutamiento-local-y-wan)
4. [Enrutamiento Dinámico (OSPF)](#4-enrutamiento-dinámico-ospf)
5. [Servicios (DHCP & NAT)](#5-servicios-dhcp--nat)
6. [Seguridad Perimetral (ACLs)](#6-seguridad-perimetral-acls)
7. [Diagnóstico y Troubleshooting](#7-diagnóstico-y-troubleshooting)

---

## 1. Fundamentos del Sistema
Gestión básica del sistema operativo y persistencia de memoria.

```text
# Cambios de modo de ejecución
enable                  # Acceso privilegiado
configure terminal      # Acceso de configuración global
exit                    # Retroceder un nivel

# Guardar la configuración en la NVRAM
write memory
# (Alternativa heredada: copy running-config startup-config)
```

---

## 2. Capa 2: LAN y Conmutación
Segmentación lógica de la red y configuración de interfaces físicas.

```text
# --- Creación de VLANs ---
vlan [número_vlan]
 name [nombre_vlan]

# --- Puertos de Acceso (Hacia equipos finales) ---
interface FastEthernet0/1
 switchport mode access
 switchport access vlan [número_vlan]

# --- Puertos Troncales (Interconexión de switches) ---
interface GigabitEthernet1/0/1
 switchport mode trunk
```

---

## 3. Capa 3: Enrutamiento Local y WAN
Conectividad inter-VLAN y enlaces geográficos.

```text
# --- Enrutamiento Inter-VLAN (Switch Multicapas) ---
ip routing  # Habilitación obligatoria del motor de capa 3

interface vlan [número_vlan]
 ip address [ip_gateway] [máscara_subred]
 no shutdown

# Convertir puerto L2 a L3 (Enrutado)
interface GigabitEthernet1/0/24
 no switchport
 ip address [ip_enlace] [máscara_subred]
 no shutdown

# --- Enlaces WAN y Ruteo Estático ---
interface Serial0/3/1
 ip address [ip_wan] [máscara_subred]
 clock rate 64000  # Solo requerido si el cable es DCE
 no shutdown

# Rutas estáticas clave
ip route [red_remota] [máscara] [siguiente_salto_wan]    # Ruta cruzada
ip route [red_local] [máscara] [ip_switch_multicapas]    # Ruta de retorno
ip route 0.0.0.0 0.0.0.0 [siguiente_salto_internet]      # Gateway of last resort
```

---

## 4. Enrutamiento Dinámico (OSPF)
Implementación de OSPF de área única con inyección de rutas y autenticación de alta seguridad.

```text
# --- Proceso y Anuncio de Redes ---
router ospf 1
 network [ip_red_wan_serial] [wildcard] area 0
 network [ip_red_lan_interna] [wildcard] area 0
 
 # Inyectar salida a internet y rutas estáticas de la LAN
 default-information originate
 redistribute static subnets

# --- Seguridad: Interfaces Pasivas ---
# Silencia el envío de "Hellos" hacia las redes locales
 passive-interface GigabitEthernet0/0
 exit

# --- Seguridad: Autenticación MD5 ---
# Se aplica directamente sobre las interfaces conectadas a otros routers
interface Serial0/3/1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 AdminOSPF123
```

---

## 5. Servicios (DHCP & NAT)
Provisión de IPs y traducción para acceso a redes públicas.

```text
# --- DHCP Relay (IP Helper) ---
interface vlan [número_vlan]
 ip helper-address [ip_servidor_dhcp_centralizado]

# --- NAT Overload (PAT) con Excepciones ---
# 1. Marcar interfaces (inside = LAN / outside = WAN)
interface GigabitEthernet0/0
 ip nat inside
interface Serial0/3/0
 ip nat outside

# 2. ACL para separar tráfico corporativo del público
ip access-list extended 100
 deny ip [red_local] [wildcard] [red_remota] [wildcard]  # Tráfico Inter-Sitio (No NAT)
 permit ip [red_local] [wildcard] any                    # Tráfico a Internet (Sí NAT)
 exit

# 3. Aplicar regla
ip nat inside source list 100 interface Serial0/3/0 overload
```

---

## 6. Seguridad Perimetral (ACLs)
Aislamiento de la red de invitados para proteger los recursos de administración y ventas.

> ⚠️ **Workaround para Bug de Packet Tracer:**
> Existe un fallo de retención al asignar ACLs nombradas en interfaces SVI. Para evitar que la configuración se rompa al reiniciar el simulador:
> * **Sucursal 1:** Utiliza ACL nombrada (`GUEST_ISOLATION`).
> * **Sucursal 2:** Utiliza ACL extendida numerada (`100`).

```text
# --- Estructura de la Lista de Acceso ---
# Permitir peticiones DHCP esenciales
permit udp any any eq bootps
# Bloquear comunicación hacia redes corporativas
deny ip [red_invitados] [máscara] [red_interna_1] [máscara]
deny ip [red_invitados] [máscara] [red_interna_2] [máscara]
# Permitir navegación a Internet
permit ip [red_invitados] [máscara] any

# --- Aplicación de la Política ---
interface vlan [número_vlan_invitados]
 ip access-group [GUEST_ISOLATION | 100] in
```

---

## 7. Diagnóstico y Troubleshooting
Comandos esenciales de verificación (Ejecutar en modo `#`).

```text
ping [ip_destino]                         # Prueba de alcance ICMP
show ip route                             # Verificar tabla de enrutamiento y convergencia
show ip interface brief                   # Estado resumido de todos los puertos físicos
show ip interface vlan [número]           # Revisar ACLs aplicadas a la VLAN
show access-lists                         # Ver reglas y conteo de paquetes bloqueados/permitidos
show ip protocols                         # Resumen del motor OSPF e interfaces pasivas
show ip ospf interface [nombre_interfaz]  # Validar temporizadores y autenticación MD5
```
---

## 8. Telefonía IP (VoIP y CME)
Configuración de Call Manager Express, puertos de voz y enrutamiento de llamadas inter-sucursal.

> 📞 **Nota sobre DHCP y TFTP:**
> Para que los teléfonos IP funcionen, el servidor DHCP debe entregar la **Opción 150 (TFTP Server)** apuntando a la IP donde el router escucha el servicio de telefonía. Además, el `ip helper-address` debe estar aplicado en la SVI de la VLAN de Voz.

```text
# --- Activación de Licencia UC (Routers Serie 2900) ---
license boot module c2900 technology-package uck9 # Activa el módulo de Comunicaciones Unificadas (requerido para VoIP)
exit                                              # Sale al modo privilegiado
write memory                                      # Guarda los cambios obligatoriamente
reload                                            # Reinicia el router para que la licencia surta efecto

# --- Configuración de Puertos para Teléfonos IP (Switch Capa 2) ---
interface FastEthernet0/1
 switchport mode access                           # Fuerza el puerto a modo acceso
 switchport access vlan [número_vlan_datos]       # Asigna la VLAN de datos para la PC conectada detrás del teléfono
 switchport voice vlan [número_vlan_voz]          # Asigna la VLAN exclusiva donde operará el teléfono IP
 spanning-tree portfast                           # Enciende el puerto de inmediato, vital para no perder el DHCP inicial

# --- Configuración del Servicio CME (Router) ---
telephony-service                                 # Ingresa al motor de Call Manager Express
 max-ephones [cantidad_max]                       # Define el límite de teléfonos físicos permitidos en la red
 max-dn [cantidad_max]                            # Define el límite de números telefónicos (líneas) a crear
 ip source-address [ip_local] port 2000           # Indica la IP y puerto (2000) donde el router escucha a los teléfonos
 auto assign 1 to [cantidad_max]                  # Asigna automáticamente las líneas creadas a los teléfonos que se conecten
exit

# --- Asignación de Extensiones ---
ephone-dn 1                                       # Crea el directorio de número (línea) 1
 number [extensión_ej_101]                        # Le asigna el número de teléfono visible a esa línea
exit
ephone-dn 2                                       # Crea el directorio de número (línea) 2
 number [extensión_ej_102]                        # Le asigna el número de teléfono visible a esa línea
exit

# --- Enrutamiento de Llamadas por WAN (Dial-Peers) ---
dial-peer voice [id_regla_ej_200] voip            # Crea una regla de ruteo para enviar llamadas por la red IP
 description Llamadas hacia Sucursal Remota       # Etiqueta descriptiva de la regla
 destination-pattern [patrón_de_marcado]          # Define qué números activa la regla (Ej: "2.." atrapa del 200 al 299)
 session target ipv4:[ip_router_remoto]           # Define a qué IP enviar el paquete de señalización de la llamada
exit
```
