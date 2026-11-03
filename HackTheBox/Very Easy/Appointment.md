# HTB: Appointment 🟣

| Parámetro | Detalle |
| :--- | :--- |
| **Plataforma** | Hack The Box (Starting Point - Tier 1) |
| **Dificultad** | Muy Fácil |
| **Dificultad Comunidad** | 🟣 Very Easy |
| **Sistema Operativo** | Linux |
| **Dirección IP** | 10.129.31.182 |

---

**Appointment** es una máquina de dificultad **Muy Fácil** orientada a introducir conceptos básicos de **Inyección SQL (SQLi)** mediante la evasión de un panel de autenticación web.

---

## 📝 Índice de Contenidos

* [🔍 1. Enumeración y Reconocimiento](#-1-enumeración)
* [💣 2. Explotación y Acceso Inicial](#-2-explotación)
* [📝 3. Conclusiones y Mitigación](#-3-conclusiones-y-mitigación)

---

## 🔍 1. Enumeración

### Escaneo de Red
Realizamos un escaneo de puertos con Nmap sobre la IP `10.129.31.182` para detectar servicios activos:

```bash
nmap -p- --open --min-rate 5000 -vvv -sS -n -Pn 10.129.31.182 -oG allPorts
extractPorts allPorts
```
