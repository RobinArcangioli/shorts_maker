# Shorts Maker — feuille de route du POC

## Base et état initial

- Dépôt de base : [mutonby/openshorts](https://github.com/mutonby/openshorts), dépôt officiel uniquement.
- Branche source : `main`.
- Commit audité : `baf66a1fa327ecbde384334544a2df579317d0f9` (25 octobre 2026).
- Branche de travail : `feat/shorts-maker-poc`.
- Périmètre : import de vidéos parlées en français ou en anglais, transcription locale, sélection locale de passages, rendu vertical et téléchargement.

La phase 0 a confirmé que le dépôt possède déjà la plupart des composants nécessaires. Les choix de modèle et les limites de ressources doivent rester configurables. Les contraintes du VPS 4 Go sont une cible de mesure, pas une limite produit.

## Phases

### Phase 0 — audit et préparation

**État : audit effectué, validation partielle.** La base, les modules, les licences et les écarts principaux sont consignés dans `docs/POC_ARCHITECTURE.md`. Les tests unitaires n’ont pas été lancés (`pytest` n’est pas installé). Le backend et FFmpeg sont maintenant construits dans Docker ; Ollama tourne également en conteneur.

### Phase 1 — environnement local et IA

Préparer l’environnement de développement et le service Ollama, retenir `qwen2.5:1.5b` comme point de départ expérimental, puis confirmer que l’API OpenAI-compatible et le modèle répondent. Configurer faster-whisper en `base`, CPU, `int8`, avec changement de modèle possible par variables d’environnement. Garder une concurrence vidéo à 1 pour le profil VPS initial.

**État : validée pour la construction et le démarrage local.** Ollama est sain et Qwen a répondu en JSON via son API OpenAI-compatible sur le réseau Compose. Le backend s’est construit après le passage des sources APT Debian en HTTPS, puis son healthcheck a répondu `{"status":"ready"}` sur le port local 8002 (les ports 8000 et 8001 étaient déjà utilisés). La transcription effective, le pipeline vidéo et la mesure de mémoire combinée restent à faire. Voir `docs/POC_RUNTIME.md`.

### Phase 2 — premier Short de bout en bout

À partir d’une vidéo française parlée de test autorisée, exécuter upload, transcription, sélection locale, découpe, recadrage et export. Vérifier le MP4, les timestamps et le journal de génération. Ne pas conclure sur la qualité sans visionner le résultat.

### Phase 3 — fiabilisation de la sélection

Évaluer la couverture des fenêtres, le classement global, la diversité, les timestamps et les réponses JSON. Corriger uniquement les problèmes observés et conserver les sorties brutes nécessaires à la comparaison.

### Phase 4 — export 1080p et sous-titres

Garder la définition de sortie configurable et fixer le profil POC à 1080 × 1920, H.264/AAC, MP4, conformément au minimum de livraison du moteur OpenShorts. Ne pas réduire la définition au seul motif d’économiser les ressources ; mesurer le coût sur le VPS. Réutiliser la transcription pour les sous-titres et vérifier les modes activé et désactivé.

### Phase 5 — parcours web minimal

Réutiliser le dashboard et le flux d’import existants. Présenter les choix de durée cible, de nombre maximal (1 à 3) et de sous-titres, ainsi que l’état, la prévisualisation et le téléchargement. Éviter une refonte générale de l’interface.

### Phase 6 — mesures sur VPS

Déployer seulement les services nécessaires, limiter les traitements lourds simultanés à un, et mesurer temps, mémoire maximale et stockage. Si 4 Go ne suffisent pas, identifier l’étape fautive et documenter les ressources nécessaires sans réduire silencieusement la qualité.

### Phase 7 — évaluation qualité

Suivre le protocole de `docs/POC_BENCHMARK.md` à créer lorsque le pipeline produit des sorties réelles. Évaluer les clips visionnés, le taux d’échec et les ressources séparément. La décision de poursuivre le produit se fonde sur les résultats observés ; aucun développement hors POC n’est engagé avant cette décision.

## Prochain chemin le plus court

1. Utiliser le runtime Docker Compose validé, avec FFmpeg et les dépendances OpenShorts.
2. Démarrer Ollama avec son API accessible depuis le backend et charger le modèle configuré.
3. Configurer faster-whisper `base`/CPU/`int8` et Ollama `qwen2.5:1.5b` via l’environnement.
4. Fournir une vidéo parlée de test autorisée, de 15 minutes maximum.
5. Réutiliser `/api/process` et `main.py` pour générer 1 à 3 passages verticaux.
6. Inspecter le MP4 avec ffprobe et le visionner avant de noter sa qualité.

## Règles de sortie de phase

Chaque phase rapporte ce qui a été réellement modifié, les commandes et résultats de vérification, les validations manuelles effectuées ou en attente, puis un commit ciblé. Aucun test ni visionnage ne doit être déclaré réussi sans exécution réelle. Les commits restent locaux tant qu’aucun push n’est demandé.
