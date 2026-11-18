# Mini SOC en la nube — Versión local (TiendaCR)

Proyecto 2 · ITI-922 Seguridad de TI II · Prof. Marcos C.

Integrantes: **Ana María Arias Campos** y **Allyson Sequeira Solís**

## Descripción

Mini SOC construido para la tienda en línea ficticia **TiendaCR**. El entorno se desplegó de forma 100% local en VirtualBox (sin nube), con tres máquinas virtuales en una red solo anfitrión: un servidor objetivo con una aplicación web vulnerable, un servidor SIEM (Wazuh) y una máquina atacante (Kali Linux). El firewall de cada máquina (UFW) cumple el rol del "grupo de seguridad" que pide el enunciado.

## Arquitectura

| Máquina | Rol | SO / software | IP (red host-only) |
|---|---|---|---|
| Objetivo | Aplicación vulnerable (Juice Shop en Docker), IDS, agente de monitoreo | Ubuntu Server 24.04 LTS, Suricata, agente Wazuh | 192.168.56.10 |
| SIEM | Servidor, indexador y panel de Wazuh | Ubuntu Server 24.04 LTS, Wazuh 4.14.10 (todo en uno) | 192.168.56.20 |
| Atacante | Origen de los ataques controlados | Kali Linux (nmap, hydra, sqlmap, nikto) | 192.168.56.30 |
| Laptop del equipo | Administración y verificación | — | 192.168.56.1 |

Cada VM tiene dos adaptadores de red: NAT (solo salida a internet) y Red solo anfitrión `vboxnet0` (192.168.56.0/24, DHCP desactivado, IPs fijas).

## Requisitos para reproducir el entorno

- VirtualBox + Extension Pack
- ISO de Ubuntu Server 24.04 LTS (x2) y de Kali Linux
- Equipo anfitrión con virtualización (VT-x) habilitada; se recomienda 16 GB de RAM o más

## Cómo reproducir el entorno

1. Crear la red solo anfitrión `vboxnet0` en VirtualBox y fijar el rango `192.168.56.1/24`, con el DHCP de esa red desactivado.
2. Crear las VMs **Objetivo** (4 GB RAM, 2 vCPU, 25 GB disco) y **SIEM** (8 GB RAM, 4 vCPU, 50 GB disco) con Ubuntu Server 24.04 LTS, e importar/instalar **Atacante** con Kali Linux (4 GB RAM, 2 vCPU).
3. En cada VM, configurar el adaptador 1 en NAT (DHCP) y el adaptador 2 en Red solo anfitrión con IP estática según la tabla de arquitectura.
4. Endurecer SSH en Objetivo y SIEM: autenticación solo por llave pública (ed25519), `PasswordAuthentication no`, `PermitRootLogin no`. Ver `config/ssh/` y las capturas en `evidence/`.
5. Desplegar la aplicación vulnerable (Juice Shop) en un contenedor Docker en el Objetivo, publicada únicamente en `192.168.56.10:3000` (nunca en `0.0.0.0`).
6. Instalar y configurar Suricata en el Objetivo, escuchando en la interfaz host-only, con las reglas de `config/suricata/local.rules`.
7. Instalar Wazuh (todo en uno) en el SIEM y enrolar el agente del Objetivo. Activar los módulos de registros de autenticación, integridad de archivos (FIM) y detector de vulnerabilidades.
8. Configurar UFW en ambas VMs con denegación por defecto y las reglas de la tabla siguiente.
9. Preparar Kali con `nmap`, `hydra`, `sqlmap` y `nikto` actualizados.
10. Tomar una instantánea (snapshot) **antes-de-atacar** en las tres VMs antes de ejecutar cualquier ataque.

## Control de acceso (UFW)

**Objetivo**

| Puerto | Acción | Origen permitido |
|---|---|---|
| 22/tcp | ALLOW IN | 192.168.56.1 |
| 3000/tcp | ALLOW IN | 192.168.56.1 y 192.168.56.30 |
| 1514/udp (saliente) | ALLOW OUT | hacia 192.168.56.20 |

**SIEM**

| Puerto | Acción | Origen permitido |
|---|---|---|
| 22/tcp | ALLOW IN | 192.168.56.1 |
| 443/tcp | ALLOW IN | 192.168.56.1 |
| 1514/udp | ALLOW IN | 192.168.56.10 |
| 1515/tcp | ALLOW IN | 192.168.56.10 |

Política por defecto en ambas VM: `deny (incoming)`, `allow (outgoing)`.

## Reglas propias

**Suricata** (`config/suricata/local.rules`):
- `sid:2000005` — detecta la herramienta sqlmap por su User-Agent.
- `sid:2000006` — detecta patrón de inyección SQL (`UNION`/`SELECT`) en la URL.
- `sid:2000007` — detecta fuerza bruta contra `/rest/user/login` (umbral 6 intentos / 60 s).

**Wazuh** (`config/wazuh/local_rules.xml`):
- `id:100100`, nivel 10 — múltiples fallos de login en la aplicación, asociada a la técnica MITRE T1110.

## Ataques ejecutados

| # | Ataque | Comando | Técnica MITRE |
|---|---|---|---|
| 1 | Reconocimiento (escaneo de puertos) | `nmap -sS -sV -p- -T4 --max-retries 1 192.168.56.10` | T1595 |
| 2 | Fuerza bruta (SSH y login de la app) | `hydra -l user -P rockyou.txt ssh://192.168.56.10` y `hydra ... /rest/user/login` | T1110 |
| 3 | Explotación web (inyección SQL) | `sqlmap -u "http://192.168.56.10:3000/rest/products/search?q=test" --batch --level=2` | T1190 |
| 4 | A elección del equipo (modificación de archivos) | **Pendiente de ejecutar** | T1565 |

## Respuesta automática

Wazuh **Active Response** (`firewall-drop`) ligada a la regla `100100`: al superar el umbral de fuerza bruta, bloquea automáticamente la IP atacante en el firewall del Objetivo. Evidencia en `evidence/` (eventos "Host Blocked by firewall-drop Active Response").

## Estructura del repositorio

```
.
├── README.md
├── docs/
│   └── P1_AriasAnaMaria_SequeiraAllyson.pdf        # informe técnico 
└── evidence/                         # capturas de pantalla numeradas por sección
```

## Seguridad y buenas prácticas

- Este repositorio **no contiene** contraseñas, llaves privadas ni tokens. Revisar antes de cada `push`.
- Las IP mostradas corresponden a una red de laboratorio privada (`192.168.56.0/24`); no son direcciones públicas.
- Todos los ataques se ejecutaron únicamente contra la infraestructura propia del equipo.

## Uso de herramientas de IA
Se utilizó un asistente de IA (Claude, Anthropic) como apoyo para redactar y dar formato al informe técnico (fases 1 y 2) a partir de la investigación y las evidencias generadas por el equipo, y para estructurar este README. El análisis técnico, los ataques, las reglas y la configuración del laboratorio fueron ejecutados por los integrantes del equipo.