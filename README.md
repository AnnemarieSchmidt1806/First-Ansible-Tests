## Einrichtung

Snipe-IT läuft in einem Docker-Container als Anwendung, in einem zweiten Container ist die MariaDB-Datenbank. Diese werden in der `docker-compose.yml` definiert. Die dazugehörigen Umgebungsvariablen befinden sich in der `.env`-Datei. Hier verweisen einige Variablen auf vertrauliche Informationen, wie Passwörter. Diese sind in der `encrypted_vars`-Datei zu finden, welche mit Ansible Vault verschlüsselt wurde. Das zugehörige Passwort für die Entschlüsselung ist im **Zoho Vault** im **Internal-IT** Ordner zu finden. Mit dem Befehl 

```sh
ansible-vault decrypt encrypted_vars
```

kann die Datei mithilfe des Passworts entschlüsselt werden.

Die virtuelle Maschine wurde mit **Ansible** konfiguriert. Der Code ist in **Azure DevOps Repositories** im Repository `thinkport_snipeit_ansible` zu finden. Hier gibt es vier Playbooks:

- **`preparation.yml`**: Installiert ein Python-Paket, welches für die Nutzung von Ansible erforderlich ist. Beim Einrichten der VM wird dieses als Erstes ausgeführt.
- **`installation.yml`**: Führt für die erstmalige Einrichtung alle notwendigen Konfigurationen aus, damit Snipe-IT funktionieren kann.
- **`maintenance.yml`**: Aktualisiert das Paketmanagementsystem des Betriebssystems.
- **`update.yml`**: Wird ausgeführt, wenn Änderungen in `docker-compose.yml` oder `.env`-Datei durchgeführt wurden, z. B. bei Versionsupdates.

Das Ausführen eines Playbooks erfolgt mit folgendem Befehl:

```sh
ansible-playbook update.yml --ask-vault-pass
```

Hier muss dann das Passwort für die `encrypted_vars`-Datei angegeben werden, damit es ausgeführt werden kann.

---

## Architektur

Die Anwendung wird in **Azure** auf einer **virtuellen Maschine** gehostet. Die Architektur sieht wie folgt aus:

![Azure Architektur](Azure_Architektur-20240918-071916.png)

Diese befindet sich in der **Subscription** `thinkport-prod` in der **Resource Group** `rg-snipeit-prod-gwc-001`. Die Ressourcen wurden mit **Terraform** angelegt und konfiguriert. Das Terraform-Skript ist im **Azure DevOps Repository** `thinkport_snipeit_terraform` zu finden. 

Die **Network Security Group (NSG)** regelt die Ports, über die auf die VM zugegriffen werden kann:

- **Port 22**: Wird für Ansible verwendet, um Änderungen vorzunehmen.
- **Port 443 (HTTPS)**: Zugriff für Benutzer über **Azure Front Door**.
- **Port 80 (HTTP)**: Azure Front Door greift über HTTP auf die VM zu.

Die **Front Door** schützt vor Angriffen und sorgt für einen schnelleren Zugriff. Sie stellt außerdem das **SSL-Zertifikat** bereit und ermöglicht die Nutzung einer eigenen **Domain**.

Innerhalb der **VM** laufen die **Docker-Container**, auf denen die Anwendung ausgeführt wird. Die **Daten** werden auf einer **Disk** gespeichert.

---

## Backups

Der **Recovery Services Vault** macht regelmäßige Backups der Disk. Die Backup-Frequenz wird über eine **Backup Policy** gesteuert:

- **Tägliches Backup um 23:00 Uhr**, gespeichert für **7 Tage**.
- **Freitags-Backup wird 12 Wochen lang gespeichert**.

---

## Maintenance und Updates

Die Anwendung muss regelmäßig gewartet und aktualisiert werden. Hierbei gibt es verschiedene Arten von Updates:

### Updates von Snipe-IT und MariaDB

- Geringer Aufwand: In der `.env`-Datei kann die aktuelle Versionsnummer hinterlegt werden.
- Wichtig: **Prüfung der Kompatibilität** zwischen Anwendung und Datenbank (Dokumentation beachten).
- Wenn alles passt, kann das **Ansible-Playbook `update.yml`** ausgeführt werden. Dabei wird nochmal ein Backup der Datenbank gemacht und die Datenbank anschließend migriert.

```sh
ansible-playbook update.yml --ask-vault-pass
```

### Updates von Linux

- Sicherheitsupdates sind unkompliziert.
- Einfach das Playbook `maintenance.yml` ausführen:

```sh
ansible-playbook maintenance.yml --ask-vault-pass
```

Für die Wartung sollte **eine Stunde pro Monat** eingeplant werden.

Aktuell läuft die Maschine auf **Ubuntu 22.04 LTS**. Alle ca. **5 Jahre** erscheint eine neue LTS-Version. Ein Upgrade erfordert:

1. Starten einer neuen VM mit der aktuellen Version.
2. Kopieren des **bestehenden Data-Directories**.
3. Ausführen der Playbooks `preparation.yml` und `installation.yml`.

Dank **Ansible** wird der Prozess erheblich vereinfacht, bleibt aber dennoch aufwendig.

---

## Nutzung von Snipe-IT

Der **Admin-Nutzer** hat alle Rechte. Die Anmeldedaten sind im **Zoho Vault** zu finden. Mit diesem Account können:

- **User angelegt** werden.
- **Inventar importiert** werden.

---
