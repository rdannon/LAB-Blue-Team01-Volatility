# 🧠 Forensique mémoire avec Volatility 2 : analyse d'un dump Windows (MemLabs, Lab 1 « Beginner's Luck »)

![Domaine](https://img.shields.io/badge/Domaine-Forensique%20m%C3%A9moire%20%2F%20DFIR-blue)
![Outil](https://img.shields.io/badge/Outil-Volatility%202.6-orange)
![Cible](https://img.shields.io/badge/Dump-Windows%207%20SP1%20x64-0078D6)
![Format](https://img.shields.io/badge/Challenge-MemLabs%20(CTF)-green)
![OS](https://img.shields.io/badge/Analyse-Ubuntu%2018.04-E95420)

> Analyse d'un **dump de mémoire vive (RAM) Windows** avec **Volatility** : identification du profil, listage des processus (y compris cachés), historique de commandes, récupération d'un fichier depuis la mémoire, extraction des **hashes NTLM**, déchiffrement d'une archive protégée et extraction des **clés de registre**, dans le cadre du challenge **MemLabs – Lab 1**.

<!-- 📸 IMAGE : capture d'ensemble du terminal Volatility (par exemple la sortie de pslist ou psscan) -->
![Vue d'ensemble du lab](images/00-vue-ensemble.png)

---

## 📑 Sommaire

1. [Présentation du projet](#1--présentation-du-projet)
2. [Objectifs](#2--objectifs)
3. [Notions théoriques](#3--notions-théoriques)
4. [Environnement et préparation](#4--environnement-et-préparation)
5. [Scénario du challenge](#5--scénario-du-challenge)
6. [Identification du profil du dump](#6--identification-du-profil-du-dump)
7. [Analyse des processus](#7--analyse-des-processus)
8. [Flag 1 : historique des commandes](#8--flag-1--historique-des-commandes)
9. [Flag 3 : récupération et déchiffrement d'une archive](#9--flag-3--récupération-et-déchiffrement-dune-archive)
10. [Autres plugins utiles](#10--autres-plugins-utiles)
11. [Synthèse des résultats](#11--synthèse-des-résultats)
12. [Compétences démontrées](#12--compétences-démontrées)
13. [Bonnes pratiques](#13--bonnes-pratiques)
14. [Structure du dépôt](#14--structure-du-dépôt)
15. [Ressources et crédits](#15--ressources-et-crédits)

---

## 1. 📌 Présentation du projet

Ce laboratoire fait partie de mon parcours de montée en compétences **Blue Team / DFIR**. Il fait suite à la collecte de mémoire (LiME sous Linux, Redline sous Windows) : une fois la mémoire capturée, il faut l'**analyser**. **Volatility** est l'un des frameworks les plus puissants pour cela.

Le travail consiste à jouer le rôle d'un **analyste en réponse à incident** devant un dump mémoire issu d'un ordinateur Windows qui a planté, et à y retrouver des informations cachées sous forme de **flags** (format CTF).

---

## 2. 🎯 Objectifs

| # | Objectif |
|---|----------|
| 1 | Comprendre ce qu'est Volatility et ce qu'il permet d'extraire de la mémoire |
| 2 | Installer Volatility 2 et préparer un dump mémoire pour analyse |
| 3 | Identifier le **profil** (système et architecture) du dump |
| 4 | Lister les processus, y compris les processus **cachés** |
| 5 | Récupérer l'historique des commandes et des arguments de processus |
| 6 | **Extraire un fichier** depuis la mémoire |
| 7 | Récupérer les **hashes NTLM** et les exploiter |
| 8 | Extraire le **registre** et les **variables d'environnement** |

---

## 3. 📚 Notions théoriques

### 3.1 Qu'est-ce que Volatility ?

**Volatility** est un framework **open source** d'analyse de mémoire pour la **réponse à incident** et l'**analyse de malwares**, écrit en **Python**. Il prend en charge **Windows, macOS et Linux** et permet d'extraire d'un dump mémoire : processus en cours, fichiers ouverts, informations sur les utilisateurs, hashes, etc.

Cette analyse permet de comprendre **quand** et **comment** une attaque a eu lieu et **ce qui tournait** au moment de la capture.

### 3.2 Volatility 2 et Volatility 3

| Version | Particularité |
|---|---|
| **Volatility 2** | Écrit en Python 2, très grand nombre de plugins. **Utilisé dans ce lab (v2.6).** |
| **Volatility 3** | Écrit en Python 3, plus récent |

### 3.3 Systèmes pris en charge

- **Windows** : de Windows XP à Windows Server 2016 ;
- **Linux** : noyaux 32 et 64 bits de la version 2.6.11 à 5.5 (Ubuntu, Debian, CentOS, Fedora, openSUSE…) ;
- **macOS**.

> ℹ️ Volatility **n'acquiert pas** la mémoire : l'acquisition se fait avec d'autres outils (**LiME** sous Linux, **DumpIt** sous Windows, etc.).

### 3.4 Notions clés

- **Profil** : description du système d'exploitation (version, build, architecture) à fournir à Volatility pour interpréter correctement le dump ;
- **Plugin** : module qui extrait un type d'information précis (processus, commandes, fichiers, hashes, registre…).

---

## 4. 🧪 Environnement et préparation

| Élément | Détail |
|---|---|
| Serveur d'analyse | **Ubuntu 18.04** (hébergé sur Linode, nom d'hôte `dfir`) |
| Outil | **Volatility 2.6** (disponible dans les dépôts d'Ubuntu 18.04) |
| Dump analysé | `MemoryDump_Lab1.raw` (~1 Go), **MemLabs – Lab 1** |
| Outils complémentaires | `p7zip`, `unrar`, `scp`, **CyberChef**, `base64` |

### 4.1 Le challenge MemLabs

**MemLabs** est un dépôt GitHub de **challenges de forensique mémoire au format CTF**, destiné aux étudiants, chercheurs en sécurité et joueurs de CTF. Il compte six laboratoires de difficulté croissante. Le **Lab 1 « Beginner's Luck »** (niveau facile) comporte **trois flags**.

<!-- 📸 IMAGE : page GitHub du dépôt MemLabs -->
![Dépôt MemLabs](images/01-depot-memlabs.png)

<!-- 📸 IMAGE : description du Lab 1 « Beginner's Luck » (énoncé et hash MD5) -->
![Énoncé du Lab 1](images/02-enonce-lab1.png)

### 4.2 Installation de Volatility

```bash
sudo apt-get update
sudo apt-get install volatility
```

Version installée : **Volatility Foundation 2.6**.

<!-- 📸 IMAGE : installation de Volatility et affichage de la version -->
![Installation de Volatility](images/03-installation-volatility.png)

### 4.3 Transfert et extraction du dump

Le dump est transféré depuis mon poste vers le serveur d'analyse avec `scp`, puis décompressé avec `p7zip` :

```bash
scp MemLabs_Lab1.7z root@<IP_SERVEUR>:/root/
sudo apt-get install p7zip-full
7z x MemLabs_Lab1.7z
ls -lh      # taille du dump : ~1 Go
```

<!-- 📸 IMAGE : transfert du fichier avec scp -->
![Transfert scp](images/04-transfert-scp.png)

<!-- 📸 IMAGE : extraction de l'archive et fichier MemoryDump_Lab1.raw obtenu -->
![Extraction du dump](images/05-extraction-dump.png)

### 4.4 Prise en main de l'aide et des plugins

```bash
volatility -h        # aide, options et plugins disponibles
volatility --info    # profils, espaces d'adressage, plugins par système
```

L'option `--info` liste les plugins classés par système d'exploitation (Mac, Linux, Windows) : c'est la référence pour savoir quoi utiliser selon le type de dump.

<!-- 📸 IMAGE : sortie de volatility -h -->
![Aide de Volatility](images/06-aide-volatility.png)

<!-- 📸 IMAGE : sortie de volatility --info (liste des plugins Windows) -->
![Plugins disponibles](images/07-info-plugins.png)

---

## 5. 🕵️ Scénario du challenge

> *« La machine de ma sœur a planté. Nous avons eu la chance de récupérer ce dump mémoire. Ta mission est de récupérer tous les fichiers importants du système. D'après nos souvenirs, nous avons vu apparaître une fenêtre noire avec quelque chose en cours d'exécution, alors qu'elle essayait de dessiner quelque chose. »* (énoncé résumé)

Le dump a été pris au moment même du plantage. Les indices à exploiter : **une fenêtre noire** (console), **un programme de dessin** et **des fichiers importants**.

---

## 6. 🔎 Identification du profil du dump

### 6.1 `imageinfo`

```bash
volatility -f MemoryDump_Lab1.raw imageinfo
```

Résultat : le premier profil suggéré est **`Win7SP1x64`**, soit un système **Windows 7 Service Pack 1 en 64 bits**. La commande indique aussi les espaces d'adressage, l'adresse DTB, la **date et l'heure de capture** et le fuseau horaire.

<!-- 📸 IMAGE : sortie de imageinfo avec les profils suggérés -->
![imageinfo](images/08-imageinfo.png)

### 6.2 `kdbgscan` : confirmation du profil

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 kdbgscan
```

`kdbgscan` affiche la structure de débogage du noyau Windows (liste des processus et modules chargés). Les résultats indiquent que le **KDBG** est valide (*owner tag check : True*) avec le profil **Windows 7 SP1 x64** : le profil est confirmé.

<!-- 📸 IMAGE : sortie de kdbgscan (KDBG owner tag check True) -->
![kdbgscan](images/09-kdbgscan.png)

---

## 7. ⚙️ Analyse des processus

### 7.1 `pslist` : processus actifs

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 pslist
```

La commande affiche, pour chaque processus : l'**offset**, le **nom**, le **PID**, le **PPID**, le nombre de **threads** et de **handles**, et l'**heure de démarrage**.

Observation : l'utilitaire **DumpIt**, qui a servi à créer le dump, apparaît dans la liste. Aucun processus suspect n'est visible au premier abord.

<!-- 📸 IMAGE : sortie de pslist -->
![pslist](images/10-pslist.png)

### 7.2 `psscan` : processus cachés ou terminés

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 psscan
```

`psscan` recherche les structures de processus dans la mémoire, y compris celles qui ne figurent plus dans la liste normale (processus terminés ou masqués). Il révèle ici :

- **`cmd.exe`** : la **fenêtre noire** décrite dans l'énoncé ;
- **`mspaint.exe`** : le **programme de dessin** (cohérent avec l'histoire) ;
- **`WinRAR.exe`** également en cours d'exécution.

> 💡 **Leçon :** `pslist` ne suffit pas ; comparer `pslist` et `psscan` est une technique de base pour détecter des processus cachés.

<!-- 📸 IMAGE : sortie de psscan avec cmd.exe, mspaint.exe et WinRAR.exe -->
![psscan](images/11-psscan.png)

### 7.3 `cmdline` : arguments d'un processus

Analyse du processus **WinRAR** (PID 1512) :

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 cmdline -p 1512
```

La ligne de commande montre que WinRAR manipulait le fichier **`Important.rar`**, situé dans le dossier **Documents** de l'utilisateur **Alissa Simpson**. Cette information est cruciale pour la suite.

<!-- 📸 IMAGE : sortie de cmdline pour le PID de WinRAR (chemin de Important.rar) -->
![cmdline WinRAR](images/12-cmdline-winrar.png)

---

## 8. 🚩 Flag 1 : historique des commandes

Le plugin **`consoles`** extrait l'**historique des commandes** saisies dans les consoles Windows, ce qui permet de voir ce qui a été exécuté dans `cmd.exe` (PID 1984, lancé avec des privilèges administrateur).

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 consoles
```

L'historique contient une commande qui fait apparaître une **chaîne encodée en Base64**. Le décodage se fait directement dans le terminal :

```bash
echo "<chaîne_base64>" | base64 -d
```

Le décodage révèle le **premier flag**.

> 🔒 *Les valeurs des flags ne sont volontairement pas publiées dans ce dépôt, afin de ne pas gâcher le challenge pour les autres.*

<!-- 📸 IMAGE : recherche du plugin consoles avec volatility --info | grep consoles -->
![Plugin consoles](images/13-plugin-consoles.png)

<!-- 📸 IMAGE : sortie de consoles (historique de commandes, chaîne Base64 masquée si besoin) -->
![Historique des commandes](images/14-consoles.png)

<!-- 📸 IMAGE : décodage de la chaîne Base64 (valeur du flag floutée) -->
![Décodage Base64](images/15-decodage-base64.png)

---

## 9. 🚩 Flag 3 : récupération et déchiffrement d'une archive

### 9.1 Localiser le fichier avec `filescan`

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 filescan | grep -i important.rar
```

`filescan` est un *pool scanner* d'objets fichiers : il retrouve l'**offset mémoire** de `Important.rar`.

<!-- 📸 IMAGE : sortie de filescan | grep -i important.rar avec l'offset -->
![filescan](images/16-filescan.png)

### 9.2 Extraire le fichier avec `dumpfiles`

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 dumpfiles -Q <offset> -D .
```

Le fichier extrait est renommé en **`important.rar`** (nom d'origine sur le système), puis son type est vérifié. L'extraction nécessite l'utilitaire `unrar` :

```bash
sudo apt-get install unrar
unrar x important.rar
```

<!-- 📸 IMAGE : commande dumpfiles et fichier extrait -->
![dumpfiles](images/17-dumpfiles.png)

<!-- 📸 IMAGE : renommage du fichier en important.rar et installation de unrar -->
![Renommage et unrar](images/18-renommage-unrar.png)

### 9.3 L'archive est protégée par mot de passe

À l'extraction, un message indique qu'un mot de passe est requis, avec l'indice suivant : le mot de passe est le **hash NTLM, en majuscules, du mot de passe du compte d'Alissa**.

<!-- 📸 IMAGE : message de demande de mot de passe de l'archive avec l'indice -->
![Archive protégée](images/19-archive-protegee.png)

### 9.4 Récupération des hashes avec `hashdump`

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 hashdump
```

Le plugin affiche les **hashes NTLM** des comptes locaux. La valeur recherchée est le hash NTLM du compte d'**Alissa**.

> ⚠️ Les hashes issus d'un challenge public ne sont pas sensibles, mais dans un cas réel, ces données doivent être traitées comme **hautement confidentielles**.

<!-- 📸 IMAGE : sortie de hashdump (hashes masqués) -->
![hashdump](images/20-hashdump.png)

### 9.5 Conversion en majuscules avec CyberChef

Le hash est mis en **majuscules** avec **CyberChef** (opération *To Upper case*), puis utilisé comme mot de passe pour extraire l'archive :

```bash
unrar x important.rar     # mot de passe : hash NTLM en majuscules
```

L'extraction fournit un fichier image, **`flag3.png`**.

<!-- 📸 IMAGE : CyberChef avec l'opération « To Upper case » -->
![CyberChef](images/21-cyberchef.png)

<!-- 📸 IMAGE : extraction réussie de l'archive avec flag3.png -->
![Extraction de l'archive](images/22-extraction-archive.png)

### 9.6 Visualisation de l'image

Le serveur n'ayant pas d'interface graphique, l'image est rapatriée sur mon poste avec `scp` :

```bash
scp root@<IP_SERVEUR>:/root/flag3.png .
```

L'ouverture de l'image donne le **troisième flag**.

<!-- 📸 IMAGE : commande scp de récupération du fichier -->
![scp de flag3.png](images/23-scp-flag3.png)

<!-- 📸 IMAGE : image flag3.png ouverte (valeur du flag floutée si nécessaire) -->
![Flag 3](images/24-flag3.png)

---

## 10. 🧰 Autres plugins utiles

### 10.1 Variables d'environnement : `envars`

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 envars
```

Affiche les variables d'environnement des processus (beaucoup de données).

### 10.2 Extraction du registre : `dumpregistry`

```bash
volatility -f MemoryDump_Lab1.raw --profile=Win7SP1x64 dumpregistry -D .
```

Écrit les **ruches du registre** (`SECURITY`, `DEFAULT`, `SOFTWARE`, etc.) dans le répertoire choisi ; elles peuvent ensuite être analysées sur un système Windows.

<!-- 📸 IMAGE : sortie de envars -->
![envars](images/25-envars.png)

<!-- 📸 IMAGE : commande dumpregistry et liste des fichiers de registre extraits -->
![dumpregistry](images/26-dumpregistry.png)

---

## 11. ✅ Synthèse des résultats

| Étape | Plugin / outil | Résultat |
|---|---|---|
| Identification du système | `imageinfo`, `kdbgscan` | Windows 7 SP1 x64 (`Win7SP1x64`) |
| Processus actifs | `pslist` | Aucun élément suspect visible, présence de DumpIt |
| Processus cachés | `psscan` | `cmd.exe`, `mspaint.exe` et WinRAR identifiés |
| Fichier manipulé | `cmdline` | `Important.rar` dans Documents d'Alissa Simpson |
| **Flag 1** | `consoles` + `base64 -d` | ✅ Retrouvé |
| Récupération du fichier | `filescan`, `dumpfiles` | ✅ `Important.rar` extrait de la mémoire |
| Mot de passe de l'archive | `hashdump` + CyberChef | ✅ Hash NTLM en majuscules |
| **Flag 3** | `unrar` | ✅ `flag3.png` récupéré |
| Registre et variables | `dumpregistry`, `envars` | ✅ Extraits |

> 📌 Le **flag 2** n'a pas été traité dans ce lab. La piste suivante serait d'analyser le processus **`mspaint.exe`**, identifié mais non exploré.

---

## 12. 🎓 Compétences démontrées

- **Forensique mémoire Windows** avec Volatility 2
- **Identification de profil** et validation (`imageinfo`, `kdbgscan`)
- **Détection de processus cachés** (`pslist` contre `psscan`)
- **Analyse de lignes de commande** et d'historiques de consoles
- **Extraction de fichiers depuis la mémoire** (`filescan`, `dumpfiles`)
- **Extraction et exploitation de hashes NTLM** (`hashdump`)
- **Manipulation de données** : Base64, CyberChef, archives protégées
- **Extraction de registre** et de variables d'environnement
- **Transferts sécurisés** (`scp`) et travail sur un serveur Linux distant
- **Raisonnement d'analyste** : partir de l'énoncé, formuler des hypothèses et les vérifier

---

## 13. 🛡️ Bonnes pratiques

- Toujours **identifier le profil** avant d'utiliser les autres plugins ;
- Comparer **plusieurs plugins** (`pslist` et `psscan`) pour détecter les processus cachés ;
- Travailler sur une **copie** du dump et vérifier son **hash MD5** avec celui fourni ;
- Manipuler les fichiers extraits d'un système infecté **dans un environnement isolé** ;
- Documenter chaque commande et chaque résultat ;
- Traiter hashes et mots de passe extraits comme des **données sensibles** ;
- Utiliser `volatility --info` pour retrouver les plugins disponibles.

---

## 14. 🗂️ Structure du dépôt

```text
volatility-memlabs-lab1/
├── README.md
├── images/
│   ├── 00-vue-ensemble.png
│   ├── 01-depot-memlabs.png
│   ├── ...
│   └── 26-dumpregistry.png
└── notes/
    └── commandes.md   (optionnel : récapitulatif des commandes)
```

---

## 15. 📎 Ressources et crédits

- **Volatility Foundation** : dépôt GitHub de Volatility
- **MemLabs** : challenges de forensique mémoire (Lab 1 « Beginner's Luck »)
- **CyberChef** : outil d'encodage et de transformation de données
- **Série Blue Team Training** (HackerSploit, avec Linode) : démonstration suivie pour ce laboratoire, reproduite et documentée ici

---

### 👤 Auteur

**Dannon Rock** : Réseaux, systèmes et cybersécurité
🔗 [LinkedIn](https://www.linkedin.com/in/rock-dannon)

> ℹ️ *Ce dépôt est un projet d'apprentissage basé sur un challenge public de forensique mémoire. Aucune donnée réelle n'est exploitée.*
