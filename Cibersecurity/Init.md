# Monitoreo & Seguridad TI — Roadmap
> Arquitectura de cero a operacional · Open-source first · Basado en estándares de industria
 
---

## Frameworks de Referencia

| Framework           | Descripción                                         | Uso                        |
| ------------------- | --------------------------------------------------- | -------------------------- |
| **CIS Controls v8** | 18 controles priorizados                            | Punto de partida principal |
| **NIST CSF 2.0**    | Govern, Identify, Protect, Detect, Respond, Recover | Marco estructural          |
| **MITRE ATT&CK**    | Tácticas y técnicas de atacantes reales             | Mapear defensas            |
| **ISO 27001**       | Estándar internacional SGSI                         | Meta a largo plazo         |

---

## Fase 1 — Visibilidad de Activos
**Timeframe:** Semanas 1–2
> Saber qué tienes antes de protegerlo. Sin inventario no hay seguridad.

### 1.1 Inventario de Hardware
- **Estándar:** CIS Control 1
- **Herramientas:** Lansweeper ★, OCS Inventory, Snipe-IT
- **Objetivo:** Descubrir automáticamente todos los endpoints, servidores, impresoras, switches, APs.

### 1.2 Inventario de Software
- **Estándar:** CIS Control 2
- **Herramientas:** Lansweeper ★, PDQ Inventory, GLPI
- **Objetivo:** Qué software está instalado, versiones, licencias. Base para gestión de parches.

### 1.3 Mapeo de Red
- **Estándar:** CIS Control 12
- **Herramientas:** Nmap ★, Angry IP Scanner, NetBox
- **Objetivo:** Diagrama de red, subnets, VLANs, dispositivos conectados. Exportar a NetBox como CMDB.

---

## Fase 2 — Monitoreo de Infraestructura
**Timeframe:** Semanas 3–5
> Monitorear disponibilidad, rendimiento y estado de todos los recursos TI en tiempo real.

### 2.1 Monitoreo de Red & Hosts
- **Estándar:** NIST SP 800-137
- **Herramientas:** Zabbix ★, Nagios, LibreNMS
- **Objetivo:** CPU, RAM, disco, uptime, latencia, SNMP traps. Alertas por umbral.

### 2.2 Monitoreo de Tráfico (Flow)
- **Estándar:** CIS Control 13
- **Herramientas:** ntopng ★, Grafana + InfluxDB, PRTG
- **Objetivo:** NetFlow/sFlow para ver quién habla con quién, bandwidth, conexiones inusuales.

### 2.3 Dashboards & Alertas
- **Estándar:** ISO 27001 A.12
- **Herramientas:** Grafana ★, Prometheus, PagerDuty
- **Objetivo:** Centralizar métricas, visualizar en tiempo real, escalar alertas a responsables.

---

## Fase 3 — Gestión de Logs & SIEM
**Timeframe:** Semanas 6–9
> Centralizar todos los logs para detectar incidentes, cumplir normativas y hacer forensia.

### 3.1 Centralización de Logs
- **Estándar:** NIST CSF / ISO 27001 A.12.4
- **Herramientas:** Graylog ★, Elastic Stack (ELK), Loki + Grafana
- **Objetivo:** Recolectar logs de Windows (WEF), Linux (syslog), firewalls, aplicaciones. Retención mínima 90 días.

### 3.2 SIEM
- **Estándar:** MITRE ATT&CK
- **Herramientas:** Wazuh ★, Microsoft Sentinel, Splunk Free
- **Objetivo:** Correlacionar eventos, detectar patrones de ataque, generar alertas de seguridad.

### 3.3 Threat Intelligence
- **Estándar:** STIX / TAXII
- **Herramientas:** OpenCTI ★, MISP, AlienVault OTX
- **Objetivo:** IOCs, TTPs de actores conocidos. Enriquecer alertas del SIEM.

---

## Fase 4 — Seguridad Endpoint & Red
**Timeframe:** Semanas 10–14
> Protección activa de equipos y red contra amenazas conocidas y desconocidas.

### 4.1 EDR / AV Empresarial
- **Estándar:** CIS Control 10
- **Herramientas:** Wazuh EDR ★, ClamAV, Defender for Business
- **Objetivo:** Detección y respuesta en endpoints. Prevenir malware, ransomware, movimiento lateral.

### 4.2 Firewall & Segmentación
- **Estándar:** CIS Control 12–13
- **Herramientas:** OPNsense ★, pfSense, Suricata IDS
- **Objetivo:** Segmentar red (usuarios, servidores, IoT). IDS/IPS en puntos clave. Zero trust progresivo.

### 4.3 Gestión de Vulnerabilidades
- **Estándar:** CIS Control 7
- **Herramientas:** Greenbone / OpenVAS ★, Nessus Essentials, Nuclei
- **Objetivo:** Escaneos periódicos de vulnerabilidades. Priorización por criticidad (CVSS). Ciclo de parcheo.

---

## Fase 5 — Identidad, Acceso & Procesos
**Timeframe:** Mes 4+
> Gobernar quién accede a qué, documentar procesos y prepararse para incidentes.

### 5.1 IAM / Directorio
- **Estándar:** CIS Control 5–6 / NIST IAM
- **Herramientas:** Active Directory ★, FreeIPA, Keycloak (SSO)
- **Objetivo:** Cuentas centralizadas, MFA obligatorio, mínimo privilegio, revisión periódica de accesos.

### 5.2 PAM & Secrets
- **Estándar:** CIS Control 12
- **Herramientas:** HashiCorp Vault ★, CyberArk (free tier), Teleport
- **Objetivo:** Controlar y auditar acceso privilegiado. Rotar credenciales automáticamente.

### 5.3 Respuesta a Incidentes
- **Estándar:** NIST SP 800-61 / ISO 27035
- **Herramientas:** TheHive ★, Shuffle SOAR, Playbooks documentados
- **Objetivo:** Proceso formal: Detección → Contención → Erradicación → Recuperación → Lecciones aprendidas.

---

## Stack Recomendado — Open Source First

| Capa | Herramienta Principal | Justificación |
|---|---|---|
| Activos & CMDB | Lansweeper + NetBox | Inventario automático + base de datos de configuración |
| Monitoreo Infra | Zabbix + Grafana | Cobertura SNMP/agente con dashboards flexibles |
| Logs & SIEM | Wazuh + Graylog | SIEM completo + centralización de logs. Todo open-source. |
| Vuln. Management | Greenbone (OpenVAS) | Scanner de vulnerabilidades open-source más completo |
| Firewall / IDS | OPNsense + Suricata | Firewall enterprise gratuito con IDS integrado |
| Respuesta | TheHive + Shuffle | Gestión de casos + automatización SOAR |

---

## Prioridad Inmediata (Primera Semana)

- [ ] Instalar **Zabbix** en VM — monitoreo de red y equipos en 2–3 horas
- [ ] Instalar **Wazuh** — SIEM con agentes en endpoints en ~1 día
- [ ] Levantar **NetBox** — documentar red mientras se descubre
- [ ] Escaneo inicial con **Nmap** — mapear todos los dispositivos activos
- [ ] Definir segmentos de red (VLANs mínimas: usuarios / servidores / gestión)

---

## Notas & Decisiones

> _Usar esta sección para documentar decisiones de arquitectura, excepciones y contexto específico de la empresa._

- [ ] Definir política de retención de logs
- [ ] Definir responsable de cada área
- [ ] Inventariar proveedores cloud actuales (AWS / Azure / GCP)
- [ ] Mapear sistemas críticos de negocio

---

*★ = Herramienta recomendada principal · Última revisión: 2026-03*
