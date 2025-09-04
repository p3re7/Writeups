# Mr Robot CTF

**Platform:** TryHackMe  
**Difficulty:** Medium  
**IP:** 10.10.10.xxx  

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



nmap -sC -sV -T5 10.10.32.21
gobuster dir -u http://10.10.32.21 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,html
permisos SUID: find / -perm -u=s -type f 2>/dev/null


