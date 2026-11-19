# Pal 1.4.3 — Rapport d'audit et de correctifs

Base : 1.4.2 (la 1.4.0 en est un sous-ensemble : 1.4.2 ajoute ~65 lignes dans Advisor, Engine, VM, project.yml et le workflow ; tout ce qui est corrigé ici vaut donc pour les deux versions).

**Statut honnête (règle 3 du dossier maître) : Implémenté — NON compilé, NON testé sur appareil.** Aucun compilateur Swift n'était disponible pendant la révision (relecture statique complète des 12 fichiers Swift, 6 300 lignes, + contrôle d'équilibre des accolades). Le build GitHub Actions est la première vraie validation.

## Corrections — fiabilité et perte de données
| # | Fichier | Défaut | Correctif |
|---|---|---|---|
| 1 | VM.importModelCopy | **Perte de données** : si le fichier choisi était déjà dans `Models`, il était supprimé avant d'être copié sur lui-même. | Détection du cas, pas de copie. |
| 2 | VM.importModelCopy | Copie de plusieurs Go **sur le thread principal** (interface gelée). Pas de contrôle d'espace. Ancien fichier supprimé avant copie. | Copie en tâche de fond, fichier temporaire `.part` puis remplacement atomique, contrôle d'espace libre, message de fin. |
| 3 | VM.loadConversations | Fichier de conversations illisible → liste vide → **écrasé à la sauvegarde suivante**. | Copie de sauvegarde `conversations-illisible-<date>.json` avant tout écrasement. |
| 4 | VM.deleteModelFile | Suppression possible **pendant une génération** (fichier mmappé supprimé). | Refusée pendant génération/chargement ; déchargement vérifié. |
| 5 | VM.regenerate | Pendant un chargement, la réponse précédente était supprimée puis l'envoi refusé. | Garde `!loading`. |
| 6 | VM.insertResearchContext | Écrasait le brouillon en cours de saisie. | Ajout à la suite du brouillon. |
| 7 | Library.Prefs | Valeurs de repli incohérentes avec les valeurs par défaut (police système et échelle 1.0 au lieu de serif et 0.85 de référence). | Repli = `Prefs()` officiel. |
| 8 | VM.init | Les Skills/personnages intégrés ajoutés par une mise à jour n'apparaissaient jamais. | Fusion sans toucher aux choix de l'utilisateur. |
| 9 | Library.scanSkillFolder | Fichier lu en entier avant contrôle de taille ; skill supprimé si fichier momentanément illisible. | Taille vérifiée d'abord ; skill conservé tant que le fichier existe. |
| 10 | VM.importDocument | Fichier lu en entier sans limite ; PDF entier extrait avant troncature ; échec sur texte non UTF-8. | Limite 50 Mo, PDF lu page par page avec arrêt anticipé, repli Latin-1. |

## Corrections — mémoire, crash, stabilité
| # | Fichier | Défaut | Correctif |
|---|---|---|---|
| 11 | Cfg / Engine / Réglages | Contexte réglable jusqu'à 65 536 : cache K/V de plusieurs Go → fermeture par iOS. | Plafond 16 384 (Cfg.maxContext), appliqué au chargement, à la lecture des réglages et au curseur. |
| 12 | Engine.load | Échec d'allocation du contexte = échec définitif. | Repli automatique : contexte divisé par 2 (jusqu'à 1024) avec message. |
| 13 | VM (alerte mémoire) | Alerte mémoire pendant une génération ignorée. | Si < 120 Mo disponibles : arrêt propre, réponse partielle conservée. |
| 14 | VM (arrière-plan) | Aucun traitement d'une génération au passage en arrière-plan (Metal interdit en arrière-plan). | Tâche d'arrière-plan pour le CPU ; arrêt propre si GPU actif ; sauvegarde forcée. |
| 15 | VM.explainLoadFailure | Accès sécurisé au fichier laissé ouvert après un échec de chargement. | Libéré après analyse. |
| 16 | VM.select | Re-toucher le modèle déjà chargé le déchargeait puis le relisait. | Ignoré si même modèle et mêmes réglages (`force` pour recharger / relier). |
| 17 | Engine | Pas de `deinit` : ressources non libérées si le moteur est détruit. | `deinit` ajouté. |
| 18 | Downloads | Entrées `lastTick` jamais purgées ; centaines d'appels/s vers le thread principal. | Purge + limitation à 2 mises à jour/s par téléchargement. |
| 19 | Render.MathCache | Cache d'images sans limite. | Limite de 300 entrées. |
| 20 | PalFolders | Modèles (plusieurs Go) inclus dans la sauvegarde iCloud/iTunes. | Dossier `Models` exclu des sauvegardes. |

## Corrections — performances
| # | Fichier | Défaut | Correctif |
|---|---|---|---|
| 21 | UI (liste + menu) | `ShareLink` écrivait **2 fichiers par conversation à chaque affichage** de la liste. | Export créé uniquement au toucher (feuille de partage). |
| 22 | VM.saveConversations | Encodage JSON + écriture de toutes les conversations sur le thread principal à chaque message. | File série en arrière-plan, écriture atomique, vidage avant suspension. |
| 23 | Render.MessageView | Découpage réflexion/réponse refait ~8× par message et par rafraîchissement. | Calculé une seule fois. |
| 24 | Render.RichText | Markdown de chaque message ré-analysé à chaque rafraîchissement (≈12×/s en génération). | Caches NSCache des blocs et du texte stylé. |
| 25 | Engine | Pénalité de répétition aveugle à la fin du prompt. | Derniers 64 tokens du prompt pris en compte. |
| 26 | Engine | Débit affiché à 0 pour une réponse d'un seul token. | Valeur calculée. |

## Corrections — incohérences d'interface
| # | Fichier | Défaut | Correctif |
|---|---|---|---|
| 27 | VM.finish | La lecture vocale automatique lisait aussi le bloc `<think>`, et même après un arrêt manuel. | Réponse seule, jamais après arrêt. |
| 28 | Speech | Session audio non libérée à l'arrêt de la lecture (musique restait en sourdine). | Libération à la fin réelle de la lecture. |
| 29 | Réglages | Interrupteur « mmap » sans effet (moteur : mmap toujours actif) — contraire au principe « visible ≠ opérationnel ». | Remplacé par une ligne d'information. |
| 30 | Réglages | Préréglages de prompt différents des personnages (« Maths » divergent). | Source unique : `Persona.defaults`. |
| 31 | Fichiers | Annulation du sélecteur de fichiers affichée comme une erreur. | Ignorée. |
| 32 | VM.exportFileURL | Nom de fichier : seul « / » était filtré. | Caractères interdits filtrés, longueur limitée. |
| 33 | VM.effectiveSystemPrompt | Lignes vides en tête quand le prompt de base est vide (le prompt doit rester vide par défaut). | Supprimées. |

## Téléchargements (priorité 1 du dossier)
| # | Fichier | Défaut | Correctif |
|---|---|---|---|
| 34 | Downloads | Aucune vérification d'intégrité alors que Hugging Face publie le SHA-256 des fichiers LFS. | SHA-256 vérifié par blocs avant installation ; en cas de différence : « Fichier corrompu ». Compatible avec les téléchargements déjà en cours. |
| 35 | Downloads | Écrasement non atomique du fichier final ; deux téléchargements de même nom possibles. | Remplacement atomique ; doublon refusé. |
| 36 | Downloads | Lien direct avec caractères encodés (`%20`) : double encodage. | Décodage avant reconstruction. |

Version : 1.4.3 (build 12), workflow mis à jour.

## Constatés, NON modifiés (décision ou mesure nécessaire)
- **Qwen 3.5 0.8B non ajouté** : cause non reproductible sans l'appareil. Hypothèse principale : architecture non reconnue par llama.cpp b12130 (le validateur du chargement affichera maintenant le message exact). Les correctifs 34-35 éliminent les autres causes possibles (fichier incomplet/corrompu).
- **Threads** : 4 threads sur l'A12 (2 cœurs rapides + 4 économes) peuvent ralentir la génération ; tester 2 vs 4 avec le benchmark intégré avant de changer la valeur par défaut.
- **GPU/Metal sur iPhone XR** : l'A12 n'est pas jugé compatible par `Sys.gpuCapable`, l'app tourne donc en CPU seul ; le « mode Puissance » ne peut pas utiliser Metal sur cet appareil.
- **Multimodal (mmproj/MTMD)** : non implémenté (l'app le dit honnêtement) ; recherche Web non reliée à la génération ; RAG absent.
- **Hack UINavigationController** (`viewDidLoad` redéfini dans une extension, UI.swift) : fonctionne mais fragile ; à surveiller à chaque nouvelle version d'iOS.
- **`Text + Text`** : déprécié dans iOS 26 (avertissement seulement).

## Tests à faire sur appareil
Chargement/déchargement, re-sélection du même modèle, copie d'un gros GGUF depuis Fichiers, téléchargement + SHA-256, mise en arrière-plan pendant une génération, lecture vocale (arrêt puis musique), export TXT/Markdown depuis la liste et depuis le menu, conversation longue avec formules (fluidité), relance après kill de l'app.
