# Aufgaben

Alle 35 Aufgaben wurden in dieser Reihenfolge umgesetzt.

## 1. Netzwerke, fw01 und Linux-VMs

**Ziel:** Die libvirt-Netze `wan` (NAT) und `lan` (isolated) sowie alle fünf VMs bereitstellen.

**Praxisbezug:** Eine saubere Basis mit Firewall und festen IP-Adressen ist die Grundlage jeder Serverumgebung.

**Kurze Schritte:**
- `wan` und `lan` mit `virsh` anlegen, DHCP aus, autostart.
- pfSense CE 2.9 auf fw01 installieren: WAN `192.168.100.10/24`, LAN `10.10.10.2/24`.
- LAN-Regeln: Alias `PORTS_LAN_OUT`, Anti-Lockout, ICMP. WAN ohne Inbound-Regeln.
- ipa01, ops01, app01, file01 mit statischen IPs installieren, Benutzer `sysadmin`, root gesperrt.
- Nach jeder Installation Snapshot mit `virsh snapshot-create-as`.

## 2. SSH-Key-Verteilung

**Ziel:** Schlüsselbasierter SSH-Zugriff von ops01 auf alle VMs.

**Praxisbezug:** Passwortloser Zugriff ist die Voraussetzung für Automatisierung.

**Kurze Schritte:**
- Ed25519-Schlüsselpaar auf ops01 erzeugen.
- Public Key mit `ssh-copy-id` an ipa01, app01, file01 und ops01 verteilen.
- Host Keys beim ersten Verbindungsaufbau prüfen und in `known_hosts` speichern.

## 3. Ansible-Installation und Inventory

**Ziel:** ansible-core auf ops01 installieren und ein Inventory für vier VMs anlegen.

**Praxisbezug:** Wiederholbare Aufgaben werden zentral von einem Control Node gesteuert.

**Kurze Schritte:**
- `pipx` mit `dnf` installieren, `ansible-core` und `ansible-lint` mit `pipx` einrichten.
- Collections `ansible.posix` und `community.general` mit `ansible-galaxy` installieren.
- `ansible.cfg` und YAML-Inventory mit den Gruppen `fedora` und `ubuntu` anlegen.
- Test mit `ansible all -m ansible.builtin.ping`.

## 4. Basis-Playbook

**Ziel:** Hostname, Zeitzone und Updates auf allen VMs einheitlich setzen.

**Praxisbezug:** Ein gleicher Grundzustand erleichtert Betrieb und Fehlersuche. Idempotenz verhindert unnötige Änderungen.

**Kurze Schritte:**
- `playbooks/update_hosts.yml` mit `ansible.builtin.hostname`, `community.general.timezone` (`Europe/Berlin`), `ansible.builtin.dnf` und `ansible.builtin.apt`.
- Neustart nur bei Bedarf mit `ansible.builtin.reboot`.
- Prüfung mit `ansible-lint`.
- Trockenlauf: `ansible-playbook playbooks/update_hosts.yml --check --diff -K`.
- Zweiter Lauf ohne Änderungen (`changed=0`).

## 5. Benutzer- und sudo-Verwaltung

**Ziel:** Lokale Admin-Gruppe `localadmins`, Benutzer `sysadmin` und sudo-Rechte auf allen VMs einrichten, root-Konto sperren.

**Praxisbezug:** Rechte müssen nachvollziehbar sein. Ein lokaler Notzugang bleibt unabhängig von FreeIPA erhalten.

**Kurze Schritte:**
- Variablen im Inventory unter `all` → `vars` definieren.
- `playbooks/users.yml` mit `ansible.builtin.group`, `ansible.builtin.user` und `ansible.posix.authorized_key`.
- sudoers-Datei in `/etc/sudoers.d/` mit Validierung durch `visudo -cf`.
- root-Passwort sperren.
- `ansible-lint`, `--check --diff` und zweiten Lauf prüfen.

## 6. SSH-Hardening (manuell)

**Ziel:** SSH-Server auf allen VMs absichern.

**Praxisbezug:** Passwort-Login und root-Login sind in Produktivumgebungen abgeschaltet.

**Kurze Schritte:**
- Snapshot `<vm>-pre-ssh-hardening` erstellen.
- `/etc/ssh/sshd_config.d/00-hardening.conf` mit `PermitRootLogin no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `MaxAuthTries 3` und weiteren Optionen anlegen.
- `sshd -t` ausführen und den Dienst neu laden. Eine zweite Sitzung bleibt währenddessen geöffnet.
- Zugriff mit Schlüssel testen, Passwort-Login muss abgelehnt werden.

## 7. firewalld (ipa01, ops01)

**Ziel:** Host-Firewall auf den Fedora-VMs mit SSH als einzigem offenen Dienst.

**Praxisbezug:** Eine minimale Angriffsfläche ist Standard. Weitere Ports werden später gezielt geöffnet.

**Kurze Schritte:**
- Dienst `firewalld` prüfen.
- In der Zone `FedoraServer` die Dienste `cockpit` und `dhcpv6-client` entfernen.
- `LogDenied=all` setzen und Konfiguration neu laden.
- SSH-Zugriff von ops01 testen.

## 8. chrony

**Ziel:** Zeitsynchronisation auf allen VMs.

**Praxisbezug:** Kerberos, TLS-Zertifikate und Logs benötigen eine korrekte Systemzeit.

**Kurze Schritte:**
- `chrony` installieren oder prüfen. Auf Ubuntu andere Zeitdienste deaktivieren.
- Zeitquelle `de.pool.ntp.org` mit `iburst` eintragen.
- Kontrolle mit `chronyc tracking` und `chronyc sources`.

## 9. Automatische Sicherheitsupdates

**Ziel:** Sicherheitsupdates automatisch einspielen, ohne automatischen Neustart.

**Praxisbezug:** Kritische Patches werden zeitnah installiert. Neustarts bleiben kontrollierbar.

**Kurze Schritte:**
- Fedora: `dnf5-plugin-automatic` mit `upgrade_type = security` und `apply_updates = yes`, Timer `dnf5-automatic.timer` aktivieren.
- Ubuntu: `unattended-upgrades` nur mit `-security`-Quelle, `Automatic-Reboot` auf `false`.
- Test mit `unattended-upgrade --dry-run` und `systemctl list-timers`.

## 10. auditd

**Ziel:** Änderungen an Konten, sudo und SSH-Konfiguration protokollieren.

**Praxisbezug:** Nachvollziehbare Änderungen entsprechen den Anforderungen von ISO/IEC 27001 und BSI IT-Grundschutz.

**Kurze Schritte:**
- Dienst `auditd` installieren und aktivieren.
- Log-Rotation in `/etc/audit/auditd.conf` setzen (`max_log_file`, `num_logs`).
- Regeln in `/etc/audit/rules.d/50-lab.rules` mit den Keys `identity`, `sudoers` und `sshd`.
- Laden mit `augenrules --load`, Test mit `ausearch -k sudoers`.

## 11. FreeIPA-Server

**Ziel:** FreeIPA mit integriertem DNS und CA auf ipa01 installieren.

**Praxisbezug:** Zentrale Identitätsverwaltung (LDAP, Kerberos) ersetzt lokale Benutzerkonten auf vielen Servern.

**Kurze Schritte:**
- Snapshot `ipa01-pre-freeipa` erstellen.
- FQDN und `/etc/hosts` prüfen, Pakete `freeipa-server` und `freeipa-server-dns` installieren.
- firewalld-Dienste `freeipa-4` und `dns` öffnen.
- `ipa-server-install` mit Domain `lab.internal`, Realm `LAB.INTERNAL`, `--setup-dns` und Forwarder `192.168.100.1`.
- Kontrolle mit `ipactl status` und `kinit admin`.

## 12. FreeIPA-Benutzer, Gruppen und DNS-Einträge

**Ziel:** Gruppen, Testbenutzer und DNS-Einträge in FreeIPA anlegen.

**Praxisbezug:** Rechte werden über Gruppen vergeben. Server müssen per Name auflösbar sein.

**Kurze Schritte:**
- Gruppen `sysops` und `fileusers` anlegen.
- Benutzer `alice` (beide Gruppen) und `bob` (nur `fileusers`) anlegen.
- A-Einträge mit PTR für fw01, ops01, app01 und file01 im DNS-Zone `lab.internal` erstellen.
- Kontrolle mit `ipa group-show` und `dig`.

## 13. FreeIPA-Client-Registrierung

**Ziel:** ops01, app01 und file01 in die FreeIPA-Domäne aufnehmen.

**Praxisbezug:** Erst nach der Registrierung gelten Benutzer, HBAC- und sudo-Regeln auf den Clients.

**Kurze Schritte:**
- Snapshot `<vm>-pre-ipa-client` erstellen.
- DNS auf `10.10.10.10` und Suchdomain `lab.internal` setzen.
- `freeipa-client` installieren, `ipa-client-install` mit `--mkhomedir` ausführen. chrony zeigt danach auf ipa01.
- Kontrolle mit `kinit admin`, `id alice` und `ipa host-find`.

## 14. FreeIPA HBAC- und sudo-Regeln

**Ziel:** Zugriff nach Gruppe, Host und Dienst begrenzen und sudo zentral steuern.

**Praxisbezug:** Die Standardregel `allow_all` ist für Produktivumgebungen zu offen.

**Kurze Schritte:**
- HBAC-Regel `sysops_ssh_all` (Dienste `sshd`, `sudo`, `sudo-i`, alle Hosts).
- HBAC-Regel `fileusers_ssh_file01` (nur `sshd` auf file01).
- Test mit `ipa hbactest`, danach `allow_all` deaktivieren.
- sudo-Regel `sysops_sudo_all` anlegen.
- SSSD-Dienst `sudo` und `sudoers: files sss` in `/etc/nsswitch.conf` prüfen.

## 15. FreeIPA-Client-Test

**Ziel:** Login, sudo und HBAC mit den FreeIPA-Benutzern praktisch prüfen.

**Praxisbezug:** Eine Installation allein zeigt nicht, ob die Regeln im Betrieb greifen.

**Kurze Schritte:**
- Initiales Passwort mit `kinit` ändern.
- SSH-Schlüssel für `alice` und `bob` erzeugen und mit `ipa user-mod --sshpubkey` hinterlegen.
- `alice`: Login auf ipa01, app01 und file01, `sudo -l` und `sudo id` erfolgreich.
- `bob`: Login nur auf file01, kein sudo, Zugriff auf app01 abgelehnt.
- Ergebnisse in den Logs prüfen.

## 16. LVM auf file01

**Ziel:** Volume Group und Logical Volume auf einem zusätzlichen Datenträger anlegen.

**Praxisbezug:** LVM ermöglicht flexible Speichererweiterung im laufenden Betrieb.

**Kurze Schritte:**
- Snapshot `file01-pre-lvm` erstellen, zwei 10-GB-Disks (`vdb`, `vdc`) anhängen.
- `pvcreate /dev/vdb`, `vgcreate vg_data /dev/vdb`.
- `lvcreate -L 6G -n lv_share vg_data`.

## 17. XFS, Mount und fstab

**Ziel:** Dateisystem anlegen und dauerhaft einbinden.

**Praxisbezug:** Mounts müssen einen Neustart überstehen. Einträge in `/etc/fstab` verwenden die UUID.

**Kurze Schritte:**
- Snapshot `file01-pre-xfs` erstellen.
- `mkfs.xfs -L share` auf `lv_share`, Mountpunkt `/srv/share` anlegen.
- Eintrag mit UUID in `/etc/fstab`, Prüfung mit `findmnt --verify` und `mount -a`.
- Neustart und Kontrolle mit `findmnt /srv/share`.

## 18. LVM erweitern

**Ziel:** Volume und Dateisystem im laufenden Betrieb vergrößern.

**Praxisbezug:** Speicherplatz ist in der Praxis irgendwann knapp. Eine Erweiterung ohne Ausfall ist eine Kernaufgabe.

**Kurze Schritte:**
- Snapshot `file01-pre-lvm-extend` erstellen, Testdatei anlegen.
- `pvcreate /dev/vdc`, `vgextend vg_data /dev/vdc`.
- `lvextend -r -l +100%FREE /dev/vg_data/lv_share` (XFS wächst mit).
- Größe mit `df -hT` prüfen, Testdatei vorhanden.

## 19. Samba-Installation

**Ziel:** Samba als Standalone-Dateiserver auf file01 installieren.

**Praxisbezug:** Dateifreigaben sind eine häufige Serveraufgabe. Eine sichere Grundkonfiguration ist wichtig.

**Kurze Schritte:**
- Snapshot `file01-pre-samba` erstellen, `samba` und `smbclient` installieren.
- `[global]`: `workgroup = LAB`, `server role = standalone server`, `server min protocol = SMB3`, `map to guest = never`, `bind interfaces only = yes`, Druckerfreigabe aus, `log level = 2 auth:3 smb2:3`.
- `nmbd` deaktivieren, `smbd` aktivieren.
- Prüfung mit `testparm`.

## 20. Samba-Freigabe und Berechtigungen

**Ziel:** Freigabe `share` für `fileusers` einrichten, Schreibrecht nur für `sysops`.

**Praxisbezug:** Abgestufte Rechte (lesen oder schreiben) sind in Dateiservern üblich.

**Kurze Schritte:**
- Gruppenauflösung mit `getent group` und `id alice` prüfen.
- `/srv/share` mit `root:fileusers` und Modus `2770` einrichten.
- `[share]` mit `valid users = @fileusers`, `read only = yes`, `write list = @sysops`, `create mask = 0660`, `directory mask = 2770`.
- Samba-Passwörter mit `smbpasswd -a` für `alice` und `bob` setzen, Kontrolle mit `pdbedit -L`.

## 21. Samba-Zugriffstest

**Ziel:** Zugriffsrechte der Freigabe praktisch prüfen.

**Praxisbezug:** Nur ein Test zeigt, wer lesen, schreiben oder gar nicht zugreifen darf.

**Kurze Schritte:**
- Freigabeliste mit `smbclient -L` abrufen.
- `alice` schreibt und liest. `bob` liest, Schreiben endet mit `NT_STATUS_ACCESS_DENIED`.
- Unbekannter Benutzer und anonymer Zugriff werden abgelehnt.
- Dateirechte (`fileusers`, `0660`) auf file01 und Logs unter `/var/log/samba/` prüfen.
- Testdaten löschen.

## 22. Docker Engine

**Ziel:** Docker Engine und Compose-Plugin auf app01 installieren.

**Praxisbezug:** Das offizielle Repository liefert aktuelle Versionen und Sicherheitsupdates.

**Kurze Schritte:**
- Snapshot `app01-pre-docker` erstellen, Ubuntu-Pakete wie `docker.io` entfernen.
- Offizielles Docker-apt-Repository (deb822) einrichten.
- `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin` und `docker-compose-plugin` installieren.
- `sysadmin` nicht in die Gruppe `docker` aufnehmen, Befehle laufen mit `sudo`.
- Test mit `hello-world`.

## 23. Docker-Compose-Stack

**Ziel:** Gitea und PostgreSQL als Stack mit Docker Compose betreiben.

**Praxisbezug:** Ein Webdienst mit Datenbank, persistentem Speicher und getrennten Netzen ist ein typisches Container-Szenario.

**Kurze Schritte:**
- `/opt/stack` anlegen, `.env` mit zufälligem Datenbankpasswort (`600`).
- `compose.yaml` mit `postgres:18` und `gitea/gitea:28.4.0`, feste Versionen.
- Netz `backend` (`internal: true`) und `frontend`, Named Volumes, `healthcheck` und `depends_on`.
- Keine Host-Ports, `docker compose up -d`.

## 24. Reverse Proxy

**Ziel:** Nginx als einzigen Zugangspunkt vor Gitea einrichten.

**Praxisbezug:** Ein Reverse Proxy bündelt den Zugriff und ist Grundlage für TLS.

**Kurze Schritte:**
- Service `proxy` mit `nginx:1.30.4-alpine` im Stack, Port `80:80`.
- `nginx/conf.d/app01.conf` mit `proxy_pass http://gitea:3000`, Forward-Headern und `client_max_body_size 100m`.
- Konfigurationsprüfung mit `nginx -t`, Zugriffstest mit `curl -I http://app01.lab.internal`.

## 25. TLS

**Ziel:** HTTPS mit einem Zertifikat der FreeIPA-CA einrichten.

**Praxisbezug:** Anmeldedaten dürfen nicht unverschlüsselt übertragen werden. certmonger erneuert das Zertifikat automatisch.

**Kurze Schritte:**
- Service `HTTP/app01.lab.internal` in FreeIPA anlegen.
- Zertifikat mit `ipa-getcert request` anfordern, Nginx-Reload nach Erneuerung.
- Nginx: Port 443 mit `TLSv1.2` und `TLSv1.3`, Port 80 leitet per `301` auf HTTPS um.
- Compose um Port `443:443` und Zertifikats-Volume erweitern, `INSTALL_LOCK` setzen, Gitea-Admin mit `gitea admin user create` anlegen.
- Test mit `curl -I https://app01.lab.internal`.

## 26. Prometheus

**Ziel:** Prometheus auf ops01 als systemd-Dienst betreiben.

**Praxisbezug:** Metriken sind die Basis für Überwachung und Alarme.

**Kurze Schritte:**
- Snapshot `ops01-pre-prometheus` erstellen, Systembenutzer `prometheus` anlegen.
- Release-Tarball mit SHA256-Prüfung installieren, `restorecon` ausführen.
- `prometheus.yml` mit `scrape_interval: 15s`.
- systemd-Unit mit `--storage.tsdb.retention.time=15d`, `--web.listen-address=127.0.0.1:9090` und Hardening-Optionen.
- Prüfung mit `promtool check config` und `/-/healthy`.

## 27. node_exporter (manuell)

**Ziel:** Systemmetriken aller vier VMs erfassen.

**Praxisbezug:** CPU, Speicher, Festplatte und Netzwerk sind die wichtigsten Betriebsdaten.

**Kurze Schritte:**
- Tarball mit SHA256-Prüfung auf ops01 laden, per `scp` an ipa01, app01 und file01 verteilen.
- Systembenutzer, systemd-Unit mit Hardening und Listen-Adresse pro VM (ops01 `127.0.0.1:9100`).
- firewalld auf ipa01: TCP 9100 nur von `10.10.10.20`.
- Job `node` in `prometheus.yml` ergänzen, alle Targets `up`.

## 28. Grafana und Dashboard

**Ziel:** Metriken in Grafana visualisieren.

**Praxisbezug:** Dashboards machen Lastverlauf und Engpässe sichtbar.

**Kurze Schritte:**
- Snapshot `ops01-pre-grafana` erstellen, offizielles RPM-Repository einrichten, `grafana` installieren.
- firewalld: TCP 3000 nur von `10.10.10.254`.
- Admin-Passwort ändern, Data Source Prometheus anlegen.
- Dashboard Node Exporter Full (ID `1860`) importieren.

## 29. Loki

**Ziel:** Zentrale Log-Speicherung mit 15 Tagen Aufbewahrung.

**Praxisbezug:** Zentrale Logs ersparen den Login auf jedem Server bei der Fehlersuche.

**Kurze Schritte:**
- Paket `loki` installieren, `/etc/loki/config.yml` mit `filesystem`-Speicher, `tsdb` und `retention_period: 360h`.
- firewalld: TCP 3100 aus `10.10.10.0/24`.
- Data Source in Grafana anlegen, Test mit `/ready`.

## 30. Grafana Alloy

**Ziel:** Journal-Logs aller VMs nach Loki senden.

**Praxisbezug:** Alloy ist der aktuelle Log-Agent im Grafana-Umfeld.

**Kurze Schritte:**
- Paket `alloy` mit `dnf` und `apt` auf allen VMs installieren.
- `/etc/alloy/config.alloy` mit `loki.source.journal`, Label `host` und `loki.write` an `10.10.10.20:3100`.
- Konfiguration mit `alloy fmt` prüfen, Benutzer `alloy` zur Gruppe `systemd-journal` hinzufügen.
- Abfrage in Grafana: `{host="app01"}`.

## 31. Prometheus-Alert-Regeln

**Ziel:** Vier Alarmregeln für typische Störungen.

**Praxisbezug:** Kritische Zustände müssen erkannt werden, bevor Anwender sie melden.

**Kurze Schritte:**
- `/etc/prometheus/rules/lab.yml` mit `HostDown`, `HighCpuUsage`, `LowDiskSpace` und `LowMemory`.
- `rule_files` in `prometheus.yml` eintragen.
- `promtool check rules` und `promtool check config`, danach Reload.
- Kontrolle über `/api/v1/rules`.

## 32. Monitoring-Daten- und Alarmtest

**Ziel:** Nachweis, dass Metriken, Logs und Alarme funktionieren.

**Praxisbezug:** Ein Alarm gilt erst als eingerichtet, wenn er im Test ausgelöst wurde.

**Kurze Schritte:**
- Daten im Grafana-Dashboard und Logs in Loki für alle Hosts prüfen.
- CPU-Last mit `yes` auf app01 erzeugen, `HighCpuUsage` wechselt auf `firing`.
- `node_exporter` auf file01 stoppen, `HostDown` wechselt auf `firing`.
- Last und Dienst wiederherstellen, Alarme gehen zurück auf `inactive`.
- Fehlgeschlagenen SSH-Login in Loki finden.

## 33. restic-Repository

**Ziel:** Verschlüsseltes Backup-Repository auf separater Platte von ops01.

**Praxisbezug:** Backups gehören auf ein anderes System und einen anderen Datenträger.

**Kurze Schritte:**
- Snapshot `ops01-pre-backup-disk` erstellen, 20-GB-Disk mit XFS unter `/srv/backup` einbinden (fstab mit UUID, Neustart-Test).
- Benutzer `restic` auf ops01, SSH-Schlüssel von file01 mit Einschränkungen (`no-pty` u. a.).
- Repository-Passwort in `/etc/restic/password` (`0600`) ablegen.
- `restic init` über SFTP.

## 34. restic-Job und systemd-Timer

**Ziel:** Nächtliche Datensicherung von file01 mit Aufbewahrungsregel.

**Praxisbezug:** Manuelle Backups werden vergessen. Zeitplan und Aufbewahrung machen sie zuverlässig.

**Kurze Schritte:**
- `/etc/restic/env` mit Repository und Passwortdatei (`0600`).
- `restic-backup.service` (oneshot) für `/srv/share`, `/etc/samba` und `/etc/fstab`, danach `restic forget --keep-daily 7 --keep-weekly 4 --prune` und `restic check`.
- `restic-backup.timer` täglich 02:00 mit `Persistent=true` und `RandomizedDelaySec`.
- Manueller Lauf und Prüfung mit `systemctl list-timers` und `restic snapshots`.

## 35. restic-Restore-Test

**Ziel:** Nachweis, dass ein Backup wiederherstellbar ist.

**Praxisbezug:** Ein nicht getestetes Backup ist keine Absicherung.

**Kurze Schritte:**
- Testdatei in `/srv/share` sichern und anschließend löschen.
- `restic restore latest --include` in ein temporäres Verzeichnis, Rechte und Inhalt prüfen.
- Gesamten Snapshot wiederherstellen und mit `diff -r` vergleichen.
- Repository mit `restic check` prüfen, Testdaten aufräumen.
