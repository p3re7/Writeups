#Ignite

**Platform:** Tryhackme
**Difficulty:** Easy

---

## 1. Información Inicial
- **Objetivo:** Obtener acceso a la máquina y capturar las flags de usuario y root.  
- **Herramientas iniciales:** `nmap`, `gobuster`, `curl`, `hydra`, `ssh`, `vim`, `linux priv esc scripts`.

---

## 2. Reconocimiento

### 2.1 Escaneo de puertos
Primero realizamos un escaneo básico con `nmap` para identificar los servicios abiertos:

```bash
nmap -sC -sV -oN nmap_initial 10.10.10.xxx
