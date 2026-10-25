# Shorts Maker — audit d’architecture OpenShorts

## Provenance

- Source : dépôt officiel `https://github.com/mutonby/openshorts`.
- Branche récupérée : `main`.
- Commit de base : `baf66a1fa327ecbde384334544a2df579317d0f9`.
- Licence à la racine : MIT. Le répertoire `cloud/` a sa propre licence commerciale ; il n’est pas requis pour le POC et ne doit pas être inclus dans un service hébergé tiers.
- Inspection locale effectuée le 25 octobre 2026. Aucun fork n’a été utilisé.

## Parcours existant réutilisable

```text
dashboard React
  → POST /api/process (app.py)
  → fichier validé et job asynchrone
  → main.py
      → transcribe_backends.transcribe_media (faster-whisper par défaut)
      → clip_selection.build_transcript_windows / classement / snap / déduplication
      → llm_backend.generate_json pour le scoring et le détail si LLM_BASE_URL est défini
      → découpe FFmpeg et reframe_v2 (avec fallback historique)
      → auto_caption_clip et subtitles.py, si les captions sont activées
  → métadonnées et fichiers MP4 dans output/<job_id>
  → endpoint de statut, affichage dashboard et téléchargement
```

### Composants et contrats

| Besoin | Composant officiel | Constat de phase 0 |
|---|---|---|
| API, import, tâches, résultats | `app.py` | `/api/process` accepte fichier et paramètres de génération ; tâches et nettoyage existent. Le niveau de concurrence par défaut vaut 5 et doit être ramené à 1 dans la configuration POC. La limite de fichier est codée à 2048 Mo ; elle doit être configurable et limitée à 300 Mo pour le profil initial. |
| Sélection et orchestration | `main.py` | `get_viral_clips` effectue un scoring en fenêtres puis un passage de détail, classe globalement, ajuste sur les mots et déduplique. `--format vertical` existe. Le résultat se réutilise sans réécrire le pipeline. |
| LLM local | `llm_backend.py`, `gemini_worker.py` | API OpenAI-compatible déjà prise en charge, validation Pydantic et repli JSON-schema → JSON-object → sans format. `LLM_BASE_URL`, `LLM_MODEL`, `LLM_TIMEOUT` et `LLM_SCORE_BATCH` sont déjà lus. Le modèle par défaut actuel est `llama3.1:8b`, à remplacer par configuration POC, pas dans le code. |
| Fenêtres et bornes de clips | `clip_selection.py` | Fenêtres couvrant la transcription, shortlist par score, bornes paramétrables `CLIP_MIN_SECONDS` / `CLIP_MAX_SECONDS`, ajustement aux mots et déduplication. L’API expose déjà `target_clips`, `clip_min_seconds` et `clip_max_seconds`. Une durée cible éditoriale à 15/30/45/60 secondes demandera d’adapter le formulaire et la logique de plage, sans couper les phrases pour atteindre une durée exacte. |
| Transcription | `transcribe_backends.py`, `subtitles.py` | faster-whisper est le backend par défaut ; VAD activé, timestamps des mots, langue détectée, contexte précédent désactivé et segments réutilisés pour la sélection et les sous-titres. Défaut actuel : `small`/CPU/`int8` ; le profil POC demandé `base`/CPU/`int8` peut être configuré sans changer le backend. |
| Rendu / recadrage | `reframe_v2.py`, `main.py`, `ffmpeg_utils.py` | Suivi et stratégie de cadrage existants, repli vers le moteur historique. `delivery_size` impose `DELIVERY_MIN_WIDTH = 1080` et conserve la hauteur source ; l’export 720 × 1280 exige une adaptation minimale et configurable du format de livraison. |
| Sous-titres | `main.py`, `subtitles.py` | SRT/ASS, synchronisation mot à mot et combustion FFmpeg réutilisables. La transcription existante évite un nouvel appel Whisper. Le parcours API accepte `captions`. |
| Interface | `dashboard/src/App.jsx` et composants | Upload, paramètres de génération, suivi, cartes vidéo et téléchargement existent déjà. Réduire le parcours au cas parlé et vérifier que les statuts et réglages POC sont exposés clairement. |
| Lancement | `docker-compose.yml`, `Dockerfile`, `dashboard/Dockerfile` | Compose lance backend, frontend dev et renderer. Il ne déclare pas Ollama. Le Dockerfile backend part de Python 3.11, installe les dépendances IA et FFmpeg ; le frontend utilise Node 18. La documentation amont annonce 8 Go+ pour l’auto-hébergement général. |

## Adaptations minimales attendues

1. Ajouter une configuration de déploiement Ollama, son volume de modèle et l’URL interne du backend ; garder le modèle et le contexte configurables.
2. Configurer Whisper via `WHISPER_MODEL=base`, `WHISPER_DEVICE=cpu`, `WHISPER_COMPUTE=int8` ; la valeur par défaut du dépôt reste `small` hors configuration.
3. Passer `MAX_CONCURRENT_JOBS` à 1 dans le profil POC ; ne pas remplacer le défaut général du logiciel.
4. Remplacer la limite source codée en dur par une variable et utiliser 300 Mo pour la configuration initiale ; vérifier également la durée maximale de 15 minutes.
5. Adapter la sélection pour une durée cible éditoriale et 1 à 3 sorties sans forcer un nombre fixe de clips. Le scoring et la seconde étape officiels restent en place.
6. Faire prendre en charge 720 × 1280 par le rendu, sans encode intermédiaire obligatoire à 1080p. Garder un profil de sortie configurable pour des machines futures.
7. Simplifier les réglages visibles dans le dashboard et rendre les états POC explicites. L’API actuelle expose notamment `queued`, `processing`, `completed` et `failed`; les sous-étapes `transcribing`, `analyzing`, `rendering` restent à vérifier/adapter.
8. Conserver un manifeste de diagnostic avec les modèles, la durée source, le nombre de clips, les sorties et les mesures réellement disponibles.

Ces changements sont des écarts à résoudre après l’audit ; aucun code applicatif n’a encore été modifié.

## Vérifications de phase 0

| Vérification | Résultat observé |
|---|---|
| Commit et branche | Commit de base `baf66a1fa327ecbde384334544a2df579317d0f9`, branche locale `feat/shorts-maker-poc`. |
| Python hôte | `Python 3.12.3`; le backend Docker utilise Python 3.11. |
| Node hôte | `v18.19.0`; les Dockerfiles du dashboard utilisent Node 18. |
| Docker Compose CLI | `Docker Compose version v5.6.1`. |
| Démon Docker | Inaccessible depuis cette session : permission refusée sur `/var/run/docker.sock`. Aucun conteneur n’a été démarré. |
| FFmpeg / Ollama sur l’hôte | Non disponibles (`ffmpeg` et `ollama` introuvables). |
| Pytest | Absent du Python système et du Python groupé disponible ; les tests unitaires n’ont donc pas été exécutés. |
| Dépendances applicatives | `requirements.txt` contient notamment faster-whisper 1.2.1, google-genai, FastAPI, PyTorch et les briques vision. Leur installation/construction d’image n’a pas été vérifiée faute de démon Docker et d’environnement Python projet. |
| Clonage / commandes | Les objets de la branche officielle ont été récupérés et le commit local correspond à la source auditée. Les commandes documentées amont sont `docker compose up --build` puis dashboard sur `http://localhost:5175`; exécution non vérifiable dans la session. |

## Risques mesurables à traiter

- **Mémoire du VPS 4 Go :** le modèle Whisper, le modèle Ollama, les processus FFmpeg et les composants vision peuvent se chevaucher. Concurrence à 1, mesures de mémoire et diagnostic par étape sont nécessaires. La documentation amont recommande 8 Go+ pour l’auto-hébergement général ; cela n’établit pas encore le besoin réel du chemin réduit du POC.
- **Qualité LLM :** README avertit que des modèles de 3B peuvent échouer à rendre le JSON fiable ; `qwen2.5:1.5b` est un essai à mesurer. Échecs JSON, latence et sélection doivent être conservés comme résultats, pas masqués.
- **Cadrage :** largeur minimale 1080 codée en dur ; le format final requis n’est pas fourni tel quel aujourd’hui.
- **Limites POC :** le dépôt est réglé pour des imports jusqu’à 2 Go, des clips et durées plus généraux, plusieurs traitements et davantage de fonctionnalités. Un profil POC configurable réduit ces paramètres sans en faire des restrictions produit.
- **Validation audiovisuelle :** aucun média de test n’est présent dans le workspace. Le premier résultat devra être inspecté au son et à l’image par un humain.
