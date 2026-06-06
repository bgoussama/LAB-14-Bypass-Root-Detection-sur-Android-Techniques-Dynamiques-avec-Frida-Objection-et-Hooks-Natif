
# LAB 14 : Bypass Root Detection sur Android

## Techniques dynamiques avec Frida, Objection et hooks natifs

## 1. Avertissement légal

Ce lab a été réalisé uniquement dans un cadre pédagogique et contrôlé.
Les techniques présentées servent à comprendre le fonctionnement des mécanismes de détection root dans les applications Android et à apprendre comment les auditer dynamiquement.

Ces manipulations doivent être utilisées uniquement sur des applications de test, des appareils personnels ou dans le cadre d’un audit autorisé.

---

## 2. Objectif du lab

L’objectif de ce lab est de contourner dynamiquement la détection de root dans une application Android à l’aide de plusieurs approches :

* utilisation de Frida et `frida-server` ;
* injection d’un script de test `hello.js` ;
* contournement des checks Java avec un script Frida ;
* contournement de certains checks natifs avec des hooks sur `libc` ;
* utilisation de Medusa ;
* utilisation d’Objection avec la commande `android root disable`.

L’application utilisée pour la validation est **RootBeer Sample**, qui affiche normalement l’état `ROOTED` lorsqu’elle détecte un environnement rooté.

---

## 3. Environnement utilisé

| Élément                | Description                               |
| ---------------------- | ----------------------------------------- |
| Système hôte           | Windows avec PowerShell                   |
| Appareil cible         | Émulateur Android                         |
| Outil principal        | Frida                                     |
| Version Frida          | 17.9.1                                    |
| Application de test    | RootBeer Sample                           |
| Package cible          | `com.scottyab.rootbeer.sample`            |
| Outils complémentaires | Objection, Medusa                         |
| Objectif final         | Afficher `NOT ROOTED` au lieu de `ROOTED` |

---

## 4. Vérification de l’environnement Frida

Avant de commencer le bypass, il faut vérifier que l’émulateur est bien détecté et que `frida-server` fonctionne.

Les commandes utilisées sont :

```powershell
adb devices
adb shell ps -A | findstr frida
frida-ps -Uai
```

La commande `adb devices` confirme que l’émulateur est connecté.
La commande `adb shell ps -A | findstr frida` montre que `frida-server` est lancé sur Android.
La commande `frida-ps -Uai` permet de lister les applications installées et visibles par Frida.

<img width="675" height="578" alt="image" src="https://github.com/user-attachments/assets/a5a15484-c0f0-4e4a-bb0a-af18822f7644" />


Cette capture prouve que l’environnement Frida est opérationnel et que l’application RootBeer Sample est détectable.

---

## 5. État initial de l’application RootBeer

Après lancement de l’application RootBeer Sample, l’application détecte que l’environnement est rooté.

Elle affiche l’état :

```text
ROOTED
```

Plusieurs checks sont déclenchés, par exemple :

* Root Management Apps ;
* Potentially Dangerous Apps ;
* TestKeys ;
* BusyBox Binary ;
* SU Binary ;
* Dangerous Props ;
* Root via native check ;
* Magisk specific checks.

**Image à insérer ici : Page 2 — partie supérieure**

<img width="372" height="486" alt="image" src="https://github.com/user-attachments/assets/fdae1ae6-28ef-4ba4-ba1b-bf1bc825f238" />



---

## 6. Test d’injection simple avec Frida

Avant de lancer un script complexe, un test simple a été réalisé avec `hello.js`.

Le script permet uniquement de vérifier que Frida peut s’injecter correctement dans l’application cible.

Commande utilisée :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l hello.js
```

Résultat obtenu :

```text
[+] Script injecté : Java.perform OK
```

**Image à insérer ici : Page 2 — partie inférieure**

<img width="667" height="223" alt="image" src="https://github.com/user-attachments/assets/396a4e86-8163-425f-abc4-0f325166b29c" />


Cette étape confirme que l’injection Frida fonctionne avant de passer au bypass réel.

---

## 7. Bypass Java avec Frida

Un script Frida Java a ensuite été utilisé afin de neutraliser les principales méthodes de détection root.

Le script cible notamment :

* `Build.TAGS` ;
* les appels à `File.exists()` ;
* les recherches de fichiers `su` et `busybox` ;
* les appels à `Runtime.exec()` ;
* les checks RootBeer ;
* les propriétés système suspectes.

La commande utilisée est :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l bypass_root_basic_v2.js
```

Les logs affichent plusieurs hooks installés :

```text
Root bypass Java v2 started
Build.TAGS -> release-keys
SystemProperties hooks installed
RootBeer hooks installed
RootBeerNative hooks installed
File.exists hook installed
Runtime.exec hooks installed
Bypass Java installed
```

Le script intercepte également plusieurs chemins suspects :

```text
/system/bin/busybox
/system/xbin/busybox
/data/local/su
/data/local/bin/su
/data/local/xbin/su
/sbin/su
/system/bin/su
/system/xbin/su
```

<img width="665" height="440" alt="image" src="https://github.com/user-attachments/assets/678f8674-ea2c-415e-b2f2-03f3435c9e02" />


Cette capture montre que les hooks Java ont été installés et que plusieurs chemins utilisés pour détecter le root ont été bloqués.

---

## 8. Bypass natif avec Frida

Certaines applications effectuent aussi des vérifications root via du code natif en C/C++.
Pour cette raison, un second script a été utilisé afin de hooker des fonctions natives de `libc`.

Les fonctions ciblées sont :

* `open` ;
* `openat` ;
* `access` ;
* `stat` ;
* `lstat` ;
* `fopen`.

Ces fonctions sont souvent utilisées pour rechercher des fichiers liés au root comme `su`, `busybox` ou certains fichiers système.

Commande utilisée :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l bypass_root_basic_v2.js -l bypass_native.js
```

Les logs montrent :

```text
Native root bypass started
Hooked libc: open
Hooked libc: openat
Hooked libc: access
Hooked libc: stat
Hooked libc: lstat
Hooked libc: fopen
Native hooks installed
```
<img width="627" height="187" alt="image" src="https://github.com/user-attachments/assets/51924c00-f632-4e3d-97a4-b446d4938d3d" />


Cette capture montre que les hooks natifs ont été installés en complément des hooks Java.

---

## 9. Résultat après bypass

Après l’injection des hooks Java et natifs, l’application RootBeer Sample ne détecte plus l’environnement comme rooté.

L’état affiché devient :

```text
NOT ROOTED
```

Les checks affichent des résultats verts, ce qui signifie que les vérifications root sont contournées.

<img width="407" height="527" alt="image" src="https://github.com/user-attachments/assets/65aebad3-c0c5-49ef-85fd-df93f2e31b12" />


Cette capture est la validation principale du lab : l’application passe de `ROOTED` à `NOT ROOTED`.

---

## 10. Utilisation de Medusa

Medusa a également été utilisé comme outil complémentaire pour analyser l’environnement Android et automatiser certaines étapes de bypass.

L’outil détecte l’émulateur Android disponible et affiche plusieurs informations système, notamment :

* fabricant ;
* modèle ;
* version Android ;
* SDK ;
* build ID ;
* tags système.

<img width="656" height="542" alt="image" src="https://github.com/user-attachments/assets/f0e1722c-7787-40ca-8e2e-1dc3b935eede" />



Cette capture montre que Medusa détecte correctement l’émulateur Android et récupère les informations système nécessaires.

---

## 11. Utilisation d’Objection

Objection a aussi été utilisé comme surcouche de Frida afin de désactiver automatiquement certains mécanismes de détection root.

La commande utilisée dans Objection est :

```text
android root disable
```

Les logs indiquent que plusieurs checks RootBeer sont interceptés et forcés vers des valeurs non suspectes :

```text
RootBeer->detectRootManagementApps() check detected, marking as false
RootBeer->detectRootCloakingApps() check detected, marking as false
RootBeer->detectTestKeys() check detected, marking as false
RootBeer->checkForBinary() check detected, marking as false
RootBeerNative->checkForRoot() check detected, marking as false
RootBeer->checkForDangerousProps() check detected, marking as false
RootBeer->checkForMagiskBinary() check detected, marking as false
```



## 12. Résultats obtenus

| Étape                                   | Résultat |
| --------------------------------------- | -------- |
| ADB détecte l’émulateur                 | Réussi   |
| `frida-server` lancé                    | Réussi   |
| `frida-ps -Uai` liste les applications  | Réussi   |
| RootBeer détecte le root au départ      | Réussi   |
| Injection `hello.js`                    | Réussie  |
| Bypass Java Frida                       | Réussi   |
| Hooks natifs Frida                      | Réussis  |
| RootBeer affiche `NOT ROOTED`           | Réussi   |
| Medusa détecte l’appareil               | Réussi   |
| Objection désactive les checks RootBeer | Réussi   |

---

## 13. Problèmes rencontrés

### Problème 1 — Options Frida non reconnues

Certaines options comme `--no-pause` ou `--resume` n’étaient pas reconnues par la version utilisée de Frida.

La solution a été d’utiliser la commande sans ces options :

```powershell
frida -U -f com.scottyab.rootbeer.sample -l script.js
```

### Problème 2 — Compatibilité frida-server

Un problème de compatibilité peut apparaître si l’architecture du binaire `frida-server` ne correspond pas à celle de l’émulateur ou de l’application.

La vérification se fait avec :

```powershell
adb shell getprop ro.product.cpu.abi
```

### Problème 3 — Checks Java insuffisants

Certains checks root peuvent être implémentés en natif.
C’est pourquoi un script natif a été ajouté pour hooker des fonctions comme `open`, `access`, `stat` ou `fopen`.

---

## 14. Captures utilisées dans le rapport

| Figure   | Description                                            | Emplacement |
| -------- | ------------------------------------------------------ | ----------- |
| Figure 1 | Frida-server actif et liste des applications           | Section 4   |
| Figure 2 | RootBeer affiche `ROOTED` avant bypass                 | Section 5   |
| Figure 3 | Test `hello.js` injecté avec succès                    | Section 6   |
| Figure 4 | Bypass Java Frida avec hooks RootBeer                  | Section 7   |
| Figure 5 | Hooks natifs Frida sur libc                            | Section 8   |
| Figure 6 | RootBeer affiche `NOT ROOTED` après bypass             | Section 9   |
| Figure 7 | Medusa détecte l’émulateur et les informations système | Section 10  |
| Figure 8 | Objection exécute `android root disable`               | Section 11  |

---

## 15. Conclusion

Ce lab a permis de comprendre comment les applications Android détectent un environnement rooté et comment ces mécanismes peuvent être contournés dynamiquement dans un cadre d’audit autorisé.

La première phase a montré que l’application RootBeer Sample détectait correctement le root.
Ensuite, l’injection Frida a permis d’installer des hooks Java pour modifier le comportement des méthodes de détection.
Des hooks natifs ont ensuite été ajoutés pour couvrir les vérifications effectuées en C/C++.

Le résultat final montre que l’application passe de l’état `ROOTED` à l’état `NOT ROOTED`.

Enfin, Objection et Medusa ont été utilisés comme outils complémentaires pour automatiser ou faciliter certaines étapes du bypass.

Ce lab montre l’importance de tester les mécanismes de sécurité Android avec plusieurs approches : Java, natif et outils automatisés.
