# TP 7 : Installation et sécurisation d'un serveur de bases de données MariaDB

**Groupe 8 — ESGIS Cotonou — L3 IRT**

Ce dépôt contient le compte rendu du TP 7 : installation, sécurisation, gestion des utilisateurs et des privilèges, sauvegarde et restauration d'un serveur MariaDB.

## Objectifs

- Installation du serveur et du client
- Création de bases et d'utilisateurs
- Gestion des privilèges
- Sauvegarde et restauration
- Démonstration : créer une base et autoriser un compte à y accéder selon des droits précis

## Environnement

| Élément | Valeur |
|---|---|
| Système | Kali Linux (machine virtuelle VirtualBox) |
| Serveur | MariaDB 11.8.9 |
| Authentification root | `unix_socket` (utilisation avec `sudo mariadb`) |

## Contenu du dépôt

```
TP7-MariaDB/
├── README.md
├── TP7_MariaDB_Guide_complet.pdf     # guide détaillé, commande par commande
├── ecole.sql                         # sauvegarde de la base de démonstration
└── captures/                         # captures d'écran du TP
```

## Déroulement du TP

### 1. Installation et sécurisation

```bash
sudo apt update
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
sudo mariadb-secure-installation
```

Vérification que le serveur n'écoute qu'en local :

```bash
grep -r "bind-address" /etc/mysql/
sudo ss -tlnp | grep 3306
```

Résultat attendu : `127.0.0.1:3306` (le serveur n'est pas joignable depuis le réseau).

### 2. Bases et utilisateurs

```sql
CREATE DATABASE ecole;
SHOW DATABASES;
CREATE USER 'osias'@'localhost' IDENTIFIED BY 'MotDePasse#Fort1';
SELECT user, host FROM mysql.user;
```

### 3. Privilèges

```sql
GRANT SELECT, INSERT, UPDATE ON ecole.* TO 'osias'@'localhost';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'osias'@'localhost';
REVOKE UPDATE ON ecole.* FROM 'osias'@'localhost';
SHOW GRANTS FOR 'osias'@'localhost';
```

Principe appliqué : **moindre privilège**. Chaque compte reçoit uniquement les droits nécessaires à son usage.

### 4. Sauvegarde et restauration

```bash
sudo mariadb-dump ecole > ecole.sql
sudo mariadb -e "DROP DATABASE ecole;"
sudo mariadb -e "CREATE DATABASE ecole;"
sudo mariadb ecole < ecole.sql
sudo mariadb -e "USE ecole; SELECT * FROM etudiants;"
```

La base est supprimée volontairement, puis restaurée depuis `ecole.sql` : les données retrouvées prouvent que la sauvegarde est valide.

### 5. Démonstration : compte en lecture seule

```sql
CREATE USER 'lecteur'@'localhost' IDENTIFIED BY 'Lecteur2026';
GRANT SELECT ON ecole.* TO 'lecteur'@'localhost';
FLUSH PRIVILEGES;
```

Connexion avec ce compte, puis tests :

```bash
mariadb -u lecteur -p ecole
```

| Commande testée | Résultat |
|---|---|
| `SELECT * FROM etudiants;` | Autorisé |
| `INSERT INTO etudiants ...` | Refusé : `ERROR 1142 (42000)` |
| `DROP TABLE etudiants;` | Refusé : `ERROR 1142 (42000)` |
| `SHOW DATABASES;` | Seulement `ecole` et `information_schema` |

## Problèmes rencontrés

| Problème | Cause et solution |
|---|---|
| `bind-address = ...` : commande introuvable | C'est une ligne de configuration (fichier `50-server.cnf`), pas une commande. Elle était déjà active. |
| `Empty set` après `SHOW TABLES;` | La base ne contient aucune table : la recréer. |
| `mariadb -u root -p` refusé | Root utilise `unix_socket` : utiliser `sudo mariadb`. |
| `ERROR 1045` pour `lecteur` | Mauvais mot de passe ou compte inexistant : `ALTER USER` ou recréer le compte. |
| Système de fichiers en lecture seule au redémarrage de la VM | Erreur ext4 : réparation avec `fsck` (`fsck.mode=force fsck.repair=yes` dans GRUB). |

## Remarque de sécurité

Les mots de passe de ce dépôt (`MotDePasse#Fort1`, `Lecteur2026`) sont des mots de passe de démonstration, utilisés uniquement sur une machine virtuelle de test. N'utilisez jamais un mot de passe réel dans un dépôt public.

## Auteur

Koffi Osias DJIBLEVO — L3 IRT, ESGIS Cotonou
