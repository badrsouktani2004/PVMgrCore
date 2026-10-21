# PVMgr.Core - Moteur de Gestion de Rapports TestStand

![C#](https://img.shields.io/badge/C%23-.NET-blue)
![NI TestStand](https://img.shields.io/badge/NI-TestStand-yellow)
![XML](https://img.shields.io/badge/Data-XML-green)
![LabVIEW Migration](https://img.shields.io/badge/Migration-LabVIEW-red)

---

## 📖 Contexte et Objectifs du Projet

Ce projet s'inscrit dans le cadre de l'amélioration des outils d'automatisation de tests industriels du groupe. L'objectif principal est de concevoir un **nouveau moteur de gestion de rapports en C# (.NET)**, compilé sous forme de bibliothèque dynamique (DLL) et destiné à s'interfacer nativement avec le séquenceur **NI TestStand**.

Il vient remplacer un algorithme historique complexe développé sous LabVIEW, offrant des performances accrues dans la génération, l'édition et la sauvegarde dynamique de rapports de test au format XML.

---

## 🎯 Cahier des Charges et Contraintes Industrielles

Pour répondre aux exigences extrêmes de la production industrielle, ce module a été forgé autour de 5 piliers critiques :

1. **Interopérabilité inter-processus** : Communication robuste avec NI TestStand via les interfaces COM (Component Object Model). Le module est capable de récupérer les paramètres de séquence en temps réel et de traiter des files d'attente (Queues) de messages à très haute fréquence.
2. **Optimisation des ressources (CPU/RAM)** : Le logiciel s'exécutant en local sur le PC du banc de test, la minimisation des accès au disque dur (opérations I/O) est primordiale. Les documents XML sont maintenus en mémoire vive (RAM) durant leur traitement. Les écritures physiques ne se déclenchent qu'en cas de timeout d'inactivité ou à la clôture finale du test.
3. **Stabilité et Endurance temporelle** : Tolérance zéro pour les fuites de mémoire (*Memory Leaks*). Le moteur est conçu pour soutenir des tests en continu sur une durée cible de **10 heures minimum**.
4. **Scalabilité et Traitement parallèle** : L'architecture est capable d'absorber la charge générée par le test de plusieurs produits en parallèle (tests multi-sockets, ex: 3 produits testés simultanément) sans générer de goulots d'étranglement ni d'erreurs de dépassement de délai.
5. **Fidélité Architecturale** : Tout en migrant vers une architecture .NET moderne, la logique fonctionnelle du LabVIEW d'origine est strictement conservée. La synchronisation temporelle (unités en secondes) est gérée avec précision pour s'aligner parfaitement avec l'orchestrateur TestStand.

---

## 🏗️ Architecture et Structure du Code

Le code source est organisé selon les principes de séparation des responsabilités (*Separation of Concerns*).

### ⚙️ Engine : `MainOrchestrator.cs`

Ce composant est le **cœur névralgique (chef d'orchestre)** de la bibliothèque. Il remplace la boucle principale (*Main Loop*) de l'ancien programme LabVIEW et centralise toute la logique d'exécution du moteur de rapports.

#### Rôle et Méthode Principale
L'orchestrateur repose sur une méthode centrale unique : `ProcessTestStandCommandsLoop`. Cette méthode maintient le *thread* actif de manière asynchrone et coordonne tous les autres services. 

Ses responsabilités sont exécutées séquentiellement à chaque itération :
* **Gestion des Timeouts :** Appelle le `ReportManagerService` pour vérifier si des rapports inactifs doivent être sauvegardés et déchargés de la RAM pour économiser des ressources (*équivalent LabVIEW : PeriodicSave*).
* **Dépilage de la File d'Attente :** Exécute une boucle de traitement rapide (jusqu'à 100 itérations par cycle) pour lire la file d'attente TestStand afin de ne manquer aucun message sous forte charge.
* **Routage des Commandes TestStand :**
  * `post` : Localise le rapport en mémoire, l'ouvre s'il est fermé, et injecte les données au bon nœud via XPath (`XmlNodeManipulatorService`).
  * `new` : Gère la création d'un nouveau fichier physique (options : `open`, `replace`, ou `rename` avec génération d'incrément automatique).
  * `delete` : Force la sauvegarde et retire un rapport spécifique du cache actif.
* **Synchronisation et Supervision :** Vérifie les demandes d'arrêt utilisateur (`Stop`, `Abort`, `Terminate`) et met à jour les propriétés du rapport directement dans les variables TestStand.

> **💡 Point Technique - Sécurité et Interopérabilité COM**  
> La robustesse de ce composant réside dans son bloc `finally` intégré à la boucle de routage. Il garantit la libération immédiate et inconditionnelle (`Marshal.ReleaseComObject`) des pointeurs `CmdContent` et `reportContentCom`. C'est cette architecture stricte qui permet au moteur de soutenir les 10 heures de test en continu sans provoquer de fuites de mémoire.

---

### 📦 Models : Structures de Données et Transfert (POCO)

Ce dossier contient l'ensemble des classes agissant comme des conteneurs de données (POCO - *Plain Old CLR Objects*). Elles garantissent un typage fort pour le transfert d'informations entre l'API COM de TestStand et la logique métier C#, éliminant ainsi les erreurs liées au typage dynamique.

#### 1. Structures des Commandes TestStand
Ces classes mappent directement les données extraites des messages de la file d'attente (Queue) TestStand :
* **`QueueMessageResult`** : Conteneur brut de la file d'attente. Identifie l'action à réaliser (`Command` type "post", "new", etc.), l'état de la file (`IsEmpty`), et transporte la charge utile (payload) sous forme d'objet COM (`CmdContent`).
* **`NewCommandInfo`** : Typage des requêtes de création de rapports. Contient le chemin physique (`FilePath`), le gabarit XSLT (`StyleSheetPath`), le tag racine (`MainTag`), et le comportement à adopter en cas de conflit (`IfExist` : *cancel, open, replace, rename*).
* **`PostCommandData`** : Typage des requêtes d'injection de données. Définit le nœud cible via une requête `XPath`, le nom de la donnée (`DataName`), la règle d'écrasement (`IfExist`), et encapsule l'objet COM TestStand à injecter (`ReportContent`).

#### 2. Gestion du Cache et Configuration
Ces classes maintiennent l'état du moteur en mémoire vive (RAM) durant l'exécution du test :
* **`ReportInfo`** : **Classe critique pour les performances I/O.** Représente un rapport actuellement ouvert. Elle stocke l'objet XML dynamique en RAM (`DocRef`), ainsi que ses marqueurs temporels (`OpenTime`, `LastUseTime`) pour le pilotage des timeouts.
* **`Parameter`** : Configuration globale du moteur lue depuis TestStand. Détermine les seuils d'inactivité avant sauvegarde physique (`OpenTimeOut`, `UseTimeOut`) et les drapeaux d'arrêt standard (`Stop`, `Abort`).

#### 3. Objets de Transfert (DTOs) et Supervision
* **`StatusMonitor`** : Pont de sécurité avec l'exécutif TestStand. Contient les références vitales (`Context`, `Execution`, `TerminationMonitor`) permettant d'écouter les demandes d'interruption (Terminate/Abort) et de déclencher une fermeture propre du moteur C#.
* **`ReportLookupResult` & `ReportOpenResult`** : Objets de retour (DTOs) utilisés par les services pour renvoyer proprement plusieurs valeurs simultanées (le rapport ciblé, son index dans la liste globale, et les potentielles erreurs capturées).

> **⚠️ Attention - Gestion des pointeurs COM**  
> Les classes `QueueMessageResult`, `PostCommandData` et `StatusMonitor` transportent des objets de type `PropertyObject` ou des références COM. Ces propriétés nécessitent impérativement d'être détruites via `Marshal.ReleaseComObject()` par les services consommateurs une fois leur traitement terminé.

---

### 🛠️ Services (Remplacement des "Sous-VIs" LabVIEW)

La logique métier est découpée en services spécialisés, reprenant et optimisant les anciens sous-VIs :

#### 📁 Service : `FileSystemService.cs`

**Rôle global du service :**  
Ce service agit comme une couche d'abstraction (Wrapper) pour toutes les opérations d'entrée/sortie (I/O) sur le disque dur. Son but principal est de sécuriser les accès physiques aux fichiers et dossiers, en s'assurant qu'aucune erreur système (ex: problème de droits d'accès, chemin invalide) ne fasse crasher le moteur principal. Il reproduit fidèlement le comportement d'exécution par flux de données de LabVIEW.

##### Détail des méthodes :
* **`CheckIfFileExists(string path, Exception errorIn)`**
  * **Rôle :** Vérifie physiquement sur le disque si le chemin fourni pointe vers un fichier ou un dossier existant.
  * **Équivalent LabVIEW :** Sous-VI *"Check if file or Folder exists"*.
  * **Détails techniques :**
    * **Propagation d'erreur (Style LabVIEW) :** La méthode prend en paramètre un `errorIn`. Si une erreur est déjà présente en amont, la méthode est court-circuitée et retourne immédiatement. Cela mime le comportement du "Cluster d'erreur" de LabVIEW.
    * **Sécurité et Typage :** Elle utilise les classes natives `File.Exists` et `Directory.Exists`. Le retour est encapsulé dans un **Tuple C#** `(bool Exists, Exception ErrorOut)`, ce qui permet de renvoyer simultanément le résultat booléen et l'état de l'erreur, facilitant grandement le traitement côté orchestrateur.

* **`GenerateUniqueFilename(string originalPath, Exception errorIn)`**
  * **Rôle :** Implémente la logique de résolution de conflits lors de la création d'un nouveau rapport (commande `new` avec paramètre `rename`). Si le fichier cible existe déjà, la méthode génère un nouveau nom en incrémentant un suffixe numérique (ex: `rapport_01.xml`, `rapport_02.xml`).
  * **Équivalent LabVIEW :** Sous-VI *"RenameExisting"*.
  * **Détails techniques :**
    * **Manipulation de chaînes sécurisée :** Utilise la classe `Path` (`GetDirectoryName`, `GetFileNameWithoutExtension`, `GetExtension`) pour dissocier le chemin proprement, indépendamment des spécificités de l'OS.
    * **Boucle de validation :** Utilise une boucle `while (File.Exists(...))` pour tester en direct la disponibilité du nouveau nom généré. Le compteur est formaté sur deux digits minimum (`:D2`) pour un rendu visuel propre (_01, _02... _10).
    * **Failsafe :** Toute exception (par exemple, un chemin trop long ou un accès refusé) est capturée, documentée via le `Logger`, et renvoyée sous forme de variable `ErrorOut` sans interrompre le programme.

---

#### 🧠 Service : `ReportManagerService.cs`

**Rôle global du service :**  
Ce service est le gestionnaire du cycle de vie des rapports en mémoire (Cache). Pour répondre aux contraintes industrielles de performance et limiter les accès au disque dur (I/O), les rapports XML sont maintenus ouverts en RAM (`XmlDocument`). Ce service est chargé d'ouvrir ces documents à la demande (Lazy Loading), de les retrouver dans la liste globale, et surtout, de les sauvegarder et de les décharger de la mémoire intelligemment grâce à un système de *Timeouts*.

##### Détail des méthodes :
* **`ManageReportTimeouts(List<ReportInfo> reportsIn, double useTimeOut, double openTimeOut, Exception errorIn)`**
  * **Rôle :** Parcourt la liste des rapports actifs et vérifie leur durée d'ouverture totale et leur temps d'inactivité. Si un des seuils (`useTimeOut` ou `openTimeOut`) est dépassé, le document est sauvegardé sur le disque puis retiré de la mémoire RAM (`DocRef = null`).
  * **Équivalent LabVIEW :** Sous-VI *"PeriodicSave"*.
  * **Détails techniques :**
    * **Calcul temporel strict :** Utilise `(DateTime.Now - report.LastUseTime).TotalSeconds` pour respecter exactement la logique de temporisation en secondes de l'ancien code LabVIEW.
    * **Optimisation de la boucle :** Si un rapport est sauvegardé, la boucle s'interrompt immédiatement (`break`). Cela évite de bloquer le thread principal (Orchestrateur) trop longtemps si plusieurs rapports expirent en même temps, garantissant un dépilage fluide de la file d'attente TestStand.
    * **Actualisation des métadonnées :** Met à jour la taille du fichier (`FileSize`) juste après la sauvegarde physique pour renvoyer une information précise à TestStand.

* **`SaveAndCloseDocument(object docRef, string filePath)`**
  * **Rôle :** Exécute la sauvegarde physique de l'objet XML sur le disque.
  * **Équivalent LabVIEW :** Sous-VI *"SaveCloseDocument"*.
  * **Détails techniques :**
    * **Sécurité de Cast :** Vérifie que l'objet en mémoire (`docRef`) est bien un `XmlDocument` valide avant d'appeler la méthode `.Save()`. Génère un `InvalidCastException` propre si TestStand a envoyé un objet corrompu.
    * **Gestion Mémoire Managée :** Fait intéressant, les appels forcés au Garbage Collector (`GC.Collect()`) ont été consciemment retirés (commentés). C'est une excellente pratique en .NET, laissant le CLR gérer la mémoire managée de manière optimisée tout en évitant des pics de charge CPU inutiles.

* **`LookupReport(List<ReportInfo> reportsIn, string reportName, Exception errorIn)`**
  * **Rôle :** Recherche un rapport spécifique par son nom dans la liste des rapports actuellement en cache.
  * **Équivalent LabVIEW :** Sous-VI *"SelectReport"*.
  * **Détails techniques :**
    * **Recherche optimisée :** Utilise la méthode LINQ `FindIndex` pour parcourir la liste de manière performante.
    * **Gestion métier des erreurs :** Si le rapport n'existe pas, la méthode ne fait pas crasher l'application mais génère une exception métier personnalisée (`customError.Data["Code"] = 0`), renvoyée proprement dans un DTO `ReportLookupResult`.

* **`OpenReportDocument(ReportInfo reportIn, Exception errorIn)`**
  * **Rôle :** Vérifie si un document XML est déjà chargé en RAM. Si ce n'est pas le cas, le fichier est lu depuis le disque dur et monté en mémoire.
  * **Équivalent LabVIEW :** Sous-VIs *"CheckAndOpen"* + *"OpenDocument"*.
  * **Détails techniques :**
    * **Stratégie de Lazy Loading (Chargement fainéant) :** La méthode ne procède au parsing physique du XML (`xmlDoc.Load`) que si `reportIn.DocRef == null`. Cela évite des accès disques redondants et coûteux si le rapport est déjà activement utilisé.
    * **Traçabilité :** Initialise le `OpenTime` avec le `DateTime.Now` exact du montage en mémoire, ce qui permet au *PeriodicSave* de calculer le timeout d'ouverture avec précision.

---

#### 🔄 Service : `TestStandService.cs`

**Rôle global du service :**  
Ce service agit comme la **passerelle de communication (Bridge)** entre l'environnement managé C# (.NET) et l'API non-managée (COM) de NI TestStand. Son rôle est de lire les variables de séquence, de dépiler la file d'attente (Queue) TestStand, et d'extraire les données brutes pour les convertir en objets C# (POCOs) utilisables par le reste de l'application. 

##### Détail des méthodes :
* **`InitializeTerminationMonitor(SequenceContext sequenceContextIn, Exception errorIn)`**
  * **Rôle :** Attache un moniteur d'exécution au contexte TestStand actuel. Cela permet au code C# de "savoir" si l'opérateur a cliqué sur le bouton rouge *Terminate* ou *Abort* dans l'interface TestStand.
  * **Équivalent LabVIEW :** Sous-VI *"Initialisation Moniteur d'arrêt de séquence"*.
  * **Détails techniques :** Utilise la méthode native `InitTerminationMonitor()` de l'objet `Execution`. Le moniteur retourné est stocké dans le `StatusMonitor` pour être interrogé en continu par l'Orchestrateur.

* **`ReadParametersFromSequence(SequenceContext context, string lookupString, Exception errorIn)`**
  * **Rôle :** Lit dynamiquement les paramètres de configuration du moteur (Timeouts) et les drapeaux d'arrêt (`Stop`, `Abort`) depuis les variables TestStand.
  * **Équivalent LabVIEW :** Sous-VI *"GetParameters"*.
  * **Détails techniques :** 
    * **Interopérabilité :** Récupère le contexte global sous forme de `PropertyObject` pour y lire des variables typées (`GetValBoolean`, `GetValNumber`).
    * **Sécurité Mémoire :** L'objet COM `propObj` est systématiquement détruit dans le bloc `finally` via `Marshal.ReleaseComObject(propObj)`. C'est vital car lire une variable TestStand crée une nouvelle référence COM en mémoire RAM.

* **`ReadCommandFromQueue(SequenceContext contextIn, string queueName, Exception errorIn)`**
  * **Rôle :** Interroge la file d'attente TestStand de manière asynchrone pour récupérer la prochaine commande de rapport à traiter.
  * **Équivalent LabVIEW :** Sous-VI *"TSDequeue"* (Timed Structure Queue Reader).
  * **Détails techniques (Point Critique) :**
    * **Non-bloquant (Timeout) :** Utilise `queue.Dequeue(...)` avec un timeout extrêmement court (`0.01` seconde). Cela permet à la boucle C# de ne jamais rester bloquée ("gelée") si la file est vide, rendant l'application très réactive.
    * **Nettoyage Extrême :** Cette méthode crée de multiples références lourdes (`engine`, `syncManager`, `queue`, `destObj`). Le bloc `finally` de cette méthode est un modèle de robustesse : il s'assure que *chaque* interface COM est libérée individuellement. C'est ce qui garantit la stabilité sur 10 heures d'exécution.

* **`ParsePostContent(PropertyObject cmdContent, Exception errorIn)`**
  * **Rôle :** Décortique le contenu d'une commande d'injection de données (`post`) extraite de la file d'attente.
  * **Équivalent LabVIEW :** Sous-VI *"TSGetReportData"*.
  * **Détails techniques :** Extrait les chaînes de caractères (XPath, ReportName) et encapsule l'objet de données (DataContent) dans un modèle `PostCommandData`. 
  * **Choix d'architecture :** L'objet COM `ReportContent` extrait ici n'est *pas* libéré dans cette méthode. Il est conservé et transmis au `MainOrchestrator` car le moteur XML en aura besoin plus tard pour lire ses sous-propriétés. La libération est déportée au niveau supérieur.

* **`ParseNewCommandContent(PropertyObject cmdContent, Exception errorIn)`**
  * **Rôle :** Décortique le contenu d'une commande de création de rapport (`new`) extraite de la file d'attente.
  * **Équivalent LabVIEW :** Sous-VI *"TSGetNewDocData"*.
  * **Détails techniques :** Effectue un mappage simple (Mapping) des propriétés textuelles (FilePath, StyleSheetPath, IfExist) vers l'objet `NewCommandInfo`. Si le paramètre d'entrée est nul, l'erreur est capturée silencieusement et logguée sans faire planter le parseur.

---

#### 🛡️ Service : `TestStandSupervisionService.cs`

**Rôle global du service :**  
Ce service agit comme le **gardien de la sécurité (Supervision)** du banc de test. Son rôle exclusif est de surveiller en continu l'état de l'exécution de TestStand pour détecter si une demande d'arrêt manuel (boutons *Terminate* ou *Abort* cliqués par l'opérateur) a été déclenchée. Il permet à l'orchestrateur C# de s'interrompre proprement plutôt que d'être "tué" brutalement, garantissant ainsi la sauvegarde des rapports en cours.

##### Détail des méthodes :
* **`CheckTerminationStatus(StatusMonitor monitorIn, Exception errorIn)`**
  * **Rôle :** Interroge périodiquement le moteur TestStand pour savoir si l'exécution est en cours d'annulation ou de terminaison.
  * **Équivalent LabVIEW :** Sous-VI *"Supervision d'arrêt de séquence"*.
  * **Détails techniques :**
    * **Appel Dynamique (Late Binding) :** Utilise le typage `dynamic` pour invoquer la méthode `GetTerminationMonitorStatus` sur l'objet `Execution` de TestStand. Cela simplifie l'interaction avec l'API COM tout en garantissant le retour du drapeau booléen d'arrêt.
    * **Impact sur l'Orchestrateur :** Le booléen retourné (`TerminatingOrAborting`) est directement utilisé dans la boucle `while` du `MainOrchestrator`. S'il passe à `true`, le moteur sort de sa boucle, sauvegarde tous les fichiers et ferme le processus proprement.

* **`CloseTerminationMonitor(StatusMonitor monitorIn, Exception errorIn)`**
  * **Rôle :** Nettoie et ferme définitivement les accès au moniteur d'exécution à la toute fin du cycle de vie du programme.
  * **Équivalent LabVIEW :** Sous-VI *"Fermeture Moniteur d'arret de sequence"*.
  * **Détails techniques (Point Critique) :**
    * **Libération mémoire en cascade :** La méthode doit libérer deux objets COM lourds : `TerminationMonitor` et `Execution`.
    * **Architecture Failsafe (Anti-Crash) :** Le code utilise une structure `try / catch / finally` imbriquée très stricte. L'objet `Execution` est déchargé dans le bloc `finally` pour garantir que même si la libération du `TerminationMonitor` génère une erreur, l'exécution principale sera **toujours** libérée (`Marshal.ReleaseComObject(monitorIn.Execution)`). Cela empêche TestStand de rester bloqué en tâche de fond (processus fantôme) à la fin du test.

---

#### 📡 Service : `TestStandSyncService.cs`

**Rôle global du service :**  
Ce service est le **pont de synchronisation d'état** de l'application. Alors que les autres services lisent les commandes depuis TestStand, celui-ci écrit dans l'espace de variables de TestStand (`StationGlobals` ou `Locals`). Son rôle est de maintenir à jour le tableau récapitulatif des rapports ouverts, afin que l'opérateur (ou d'autres séquences) puisse connaître en temps réel la taille, le chemin et le statut des fichiers XML en cours d'édition.

##### Détail des méthodes :
* **`ParamSet_SyncWithTestStand(List<ReportInfo> reportsIn, dynamic sequenceContextIn, string lookupString, Exception errorIn)`**
  * **Rôle :** Parcourt la liste locale C# des rapports ouverts et la calque fidèlement dans un tableau de conteneurs TestStand. Il met à jour les éléments existants, crée les éléments manquants, et supprime les éléments obsolètes (les rapports qui viennent d'être fermés/sauvegardés).
  * **Équivalent LabVIEW :** Sous-VI *"SetReportsInfo"*.
  * **Détails techniques :**
    * **Création et Injection dynamiques :** La méthode utilise `propertyObject.Exists()` pour vérifier si le rapport est déjà référencé dans TestStand. Si ce n'est pas le cas, elle appelle l'API COM `InsertElements` pour agrandir le tableau TestStand dynamiquement et y insérer le nouveau rapport.
    * **Conversion Temporelle (COM/OLE) :** Les dates C# (`DateTime`) sont converties via `.ToOADate()` (OLE Automation Date). C'est une étape cruciale pour que TestStand (qui est basé sur les technologies COM) puisse interpréter correctement les timestamps (LastUseTime, OpenTime).
    * **Nettoyage inversé (Garbage Collection) :** Pour supprimer les anciens rapports de TestStand, la méthode stocke les index valides (`validIndices`), puis parcourt le tableau TestStand **à l'envers** (`for (int i = totalElementsInTestStand - 1; i >= 0; i--)`). C'est une excellente pratique algorithmique pour supprimer des éléments d'une liste dynamique sans décaler les index restants.
    * **Gestion Mémoire Intra-boucle :** La création d'objets COM se fait *à l'intérieur* d'une boucle `foreach`. Pour éviter une explosion de la RAM (Memory Leak), un bloc `try/finally` est imbriqué à l'intérieur même de la boucle pour garantir le `Marshal.ReleaseComObject` de `parentContainer` et `targetElement` à chaque itération.

---

#### 📝 Service : `XmlFileCreatorService.cs`

**Rôle global du service :**  
Ce service est responsable de l'initialisation et de la création physique des nouveaux fichiers XML sur le disque dur. Il agit comme un générateur de gabarit (Template Engine) de base lorsqu'une commande `new` (avec l'instruction de créer ou remplacer un fichier) est reçue depuis TestStand.

##### Détail des méthodes :
* **`CreateXmlFile(string path, string styleSheetPath, string mainTag, Exception errorIn)`**
  * **Rôle :** Génère la structure squelette du fichier XML (Déclaration XML, lien vers la feuille de style, balise racine) et l'écrit physiquement sur le disque.
  * **Équivalent LabVIEW :** Sous-VI *"NewDocument"*.
  * **Détails techniques :**
    * **Optimisation par Interpolation (String vs DOM) :** Plutôt que d'instancier un objet lourd `XmlDocument` en mémoire pour créer 3 lignes de code, le service utilise l'interpolation de chaînes C# (`$"{...}"`). Cette approche est extrêmement légère et rapide en termes d'allocation CPU/RAM.
    * **Écriture Atomique (I/O) :** L'utilisation de `File.WriteAllText(path, xmlContent)` est un choix sécurisé. Cette méthode gère automatiquement l'ouverture, l'écriture et la fermeture du flux de fichier (Stream), s'assurant que le fichier ne reste pas verrouillé (*Locked*) par le système d'exploitation en cas d'erreur.
    * **Intégration XSLT native :** Le service injecte dynamiquement l'instruction de traitement `<?xml-stylesheet...?>`. Cela permet au rapport XML généré d'être immédiatement mis en forme et lisible visuellement (via CSS/HTML) lorsqu'il est ouvert dans un navigateur web par un opérateur.
    * **Propagation d'erreur :** Comme pour le reste de l'architecture, la méthode vérifie `errorIn` dès la première ligne pour simuler le flux de données LabVIEW et s'interrompt si le programme est déjà en état d'échec.

---

#### 🧩 Service : `XmlNodeManipulatorService.cs`

**Rôle global du service :**  
Ce service est le **Moteur de Traduction TestStand vers XML**. Il est responsable de l'interprétation des données brutes envoyées par TestStand (`PropertyObject`) et de leur injection dynamique sous forme de nœuds dans l'arbre DOM (Document Object Model) du fichier XML en mémoire. Il gère l'ajout, le remplacement, et navigue dans les structures de données complexes (tableaux, conteneurs imbriqués).

##### Détail des méthodes :
* **`ManipulateXmlNode(XmlDocument docRef, XmlNode parentNode, dynamic reportContent, string ifExist, string dataName, Exception errorIn)`**
  * **Rôle :** Méthode "couteau suisse" qui analyse le type de la donnée TestStand (Nombre, Chaîne, Conteneur, Tableau) et génère le nœud XML correspondant au bon emplacement (défini par une requête XPath).
  * **Équivalent LabVIEW :** Cette unique méthode remplace à elle seule un VI principal (*AddDataToXML*) et ses 4 sous-VIs polymorphes (*AddNode*, *AddStringValue*, *ReplaceNode*, *ReplaceStringValue*).
  * **Détails techniques (Points Critiques) :**
    * **Algorithmique Récursive :** C'est le point fort du service. Si le nœud TestStand à injecter est un "Container" (Type 0), une référence (Type 5) ou un "Array" (Type 6), la méthode s'appelle elle-même (récursivité) pour descendre dans l'arborescence et traiter chaque sous-propriété individuellement. Cela permet de supporter des arborescences XML d'une profondeur infinie.
    * **Mappage des Types TestStand (`ValueType`) :** Le code identifie dynamiquement le type de variable via l'API COM :
      * *Type 0 (Container) :* Crée un nœud parent et lance une itération récursive sur les sous-propriétés.
      * *Types 1, 2, 3, 7 (Primitives) :* Extrait la valeur sous forme de texte pur (`GetValString`) avec des masques binaires (options `0x00`, `0x20`, `0x08`, `0xC0` pour formater les booléens, nombres, etc.) et crée un `XmlTextNode`.
      * *Type 6 (Array) :* Boucle sur la taille du tableau (par *Offset*) et injecte chaque élément. (Note de conception : la méthode bloque volontairement le remplacement direct d'un tableau entier pour des raisons de sécurité de structure XML).
    * **Nettoyage COM "Micro-Managé" (Failsafe) :** Dans ce composant, la gestion de la mémoire non-managée est poussée à l'extrême. Étant donné la récursivité et les boucles (`foreach`, `for`), des références COM sont créées à la volée des centaines de fois par seconde (`typeObjAdd`, `subProperty`, `arrayElement`). Pour éviter une explosion de la RAM (Fuite Mémoire), **chaque assignation COM est encadrée par son propre bloc `try/finally` local**, libérant l'interface COM instantanément à la fin de la boucle, avant même de passer à l'itération suivante. L'objet parent `reportContent` est également libéré en fin de méthode.
    * **Gestion des Conflits XPath :** Utilise `SelectSingleNode(dataName)` pour vérifier si le nœud existe déjà. Si la requête TestStand demande "add" mais que le nœud existe, ou demande un "replace", l'action finale (`actualCommand`) est adaptée automatiquement pour ne jamais corrompre l'arbre XML (`ReplaceChild` ou `AppendChild`).

---

### 🛠️ Utils : `Logger.cs`

**Rôle global du composant :**  
Ce composant utilitaire statique est le **système de traçage (Logging) de l'application**. Son rôle est de capturer, formater et sauvegarder toutes les exceptions et anomalies métier dans un fichier texte physique local (par défaut `C:\Temp\TestStand_PVMgr_Log.txt`). Il permet un débogage à posteriori essentiel pour la maintenance du banc de test.

##### Détail de la méthode :
* **`WriteError(string module, string message, Exception ex = null)`**
  * **Rôle :** Construit une ligne de log horodatée contenant la source de l'erreur (le module ou "Sous-VI" concerné), un message descriptif, et le détail technique de l'exception C# si elle est fournie.
  * **Équivalent LabVIEW :** Fait office de *General Error Handler* et de journalisation système.
  * **Détails techniques (Point Critique) :**
    * **Création dynamique et Horodatage :** Vérifie l'existence du dossier cible et le crée à la volée si nécessaire. Chaque ligne est horodatée à la milliseconde près (`yyyy-MM-dd HH:mm:ss.fff`) pour permettre de croiser les logs avec les rapports d'exécution TestStand.
    * **Failsafe Absolu (Anti-Crash) :** C'est la caractéristique la plus importante de cette classe. L'intégralité de la méthode est encapsulée dans un bloc `try / catch` dont le `catch` est **volontairement vide (silencieux)**. Dans un contexte industriel continu (10h de test), si le disque dur du PC devient plein ou si le fichier texte est verrouillé par un autre processus (ex: Antivirus ou bloc-notes ouvert par un opérateur), l'écriture échouera. Ce bloc silencieux garantit que l'échec de la journalisation ne fera **jamais** crasher la séquence TestStand en cours. Il vaut mieux perdre une ligne d'erreur dans un fichier texte que d'interrompre la production d'une usine.

---

## 🧠 Défis Techniques & Qualité du Code

* **Gestion Stricte de la Mémoire COM (Safe Release) :** L'interaction avec l'API Interop de TestStand génère d'innombrables références non managées (`PropertyObject`). Chaque appel est sécurisé par des blocs `try/finally` incluant `Marshal.ReleaseComObject()` pour garantir un nettoyage instantané et empêcher les crashs de saturation RAM sur le long terme.
* **Timeouts Dynamiques Intelligents :** Les documents ne sont sauvegardés physiques que si la durée d'inactivité (`UseTimeOut`) ou la durée totale d'ouverture (`OpenTimeOut`) dépasse les seuils définis, combinant ainsi haute disponibilité en RAM et sécurité des données.

---

## 📊 Intégration et Bilan des Validations

* **Intégration** : La DLL est instanciée via le **.NET Adapter** de TestStand. Le `SequenceContext` (`Step.Context`) est passé par référence à l'orchestrateur.
* **Validation sur Banc de Test** :
  * **Endurance** : Validé avec succès sur un cycle de 10 heures d'exécution ininterrompue.
  * **Stabilité Mémoire** : Empreinte RAM lissée et maîtrisée (~45 Mo en moyenne), sans aucune dérive grâce à la libération COM explicite.
  * **Fiabilité sous Charge** : 0 perte de requête XML constatée sous environnement de test parallèle (Multi-sockets).
