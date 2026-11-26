# Linux System Administration PoC

Dieses Proof of Concept (PoC) zeigt eine kleine Linux-Umgebung mit typischen Aufgaben eines Linux System Administrators. Die Umgebung läuft vollständig virtualisiert auf einem Host mit KVM/QEMU und libvirt. Alle Aufgaben wurden praktisch umgesetzt.

## Ziel

Aufbau einer produktionsnahen Umgebung mit Firewall, zentraler Benutzerverwaltung, Dateiserver, Container-Dienst, Monitoring und Backup. Die Sicherheitseinstellungen orientieren sich an den Prinzipien von ISO/IEC 27001 und BSI IT-Grundschutz.

## Technologien

- Virtualisierung: KVM/QEMU, libvirt (virsh, virt-install, qcow2)
- Betriebssysteme: pfSense CE 2.9, Fedora Server 45, Ubuntu Server 26.04 LTS
- Netzwerk und Firewall: pfSense, firewalld, chrony
- Automatisierung: Ansible (ansible-core, Installation mit pipx)
- Sicherheit: SSH-Hardening, auditd, automatische Sicherheitsupdates
- Identitätsverwaltung: FreeIPA (Kerberos, LDAP, DNS, CA), HBAC- und sudo-Regeln
- Speicher: LVM, XFS
- Dateiserver: Samba (nur Dateifreigabe)
- Container: Docker Engine, Docker Compose (Gitea, PostgreSQL, Nginx mit TLS)
- Monitoring: Prometheus, node_exporter, Grafana, Loki, Grafana Alloy
- Backup: restic

## Umfang

- 5 virtuelle Maschinen in zwei Netzen (wan, lan)
- 35 Aufgaben
- Statische IP-Adressen, kein DHCP, kein VPN
- FreeIPA: Grundinstallation, Client-Anbindung und Identitätsverwaltung
- Samba: nur Dateifreigabe, kein Active Directory Domain Controller
- Docker: nur Container-Verwaltung, kein Kubernetes und kein Docker Swarm

## Dokumentation

- [AUFGABEN.md](AUFGABEN.md): alle Aufgaben mit Ziel, Praxisbezug und kurzen Schritten
- [UMGEBUNG.md](UMGEBUNG.md): VMs, Ressourcen, IP-Plan und Netzwerktopologie
