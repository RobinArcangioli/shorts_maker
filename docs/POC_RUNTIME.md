# Runtime local du POC

Le profil `docker-compose.poc.yml` lance seulement FastAPI et Ollama. Le backend est accessible sur `http://localhost:8000` par défaut, tandis qu’Ollama reste privé au réseau Docker. Si le port 8000 est déjà utilisé, le port local peut être remplacé, par exemple avec `SHORTS_MAKER_PORT=8002`. Les données Ollama, les imports, les exports et le cache Whisper sont conservés dans des volumes nommés.

```bash
SHORTS_MAKER_PORT=8002 docker compose -f docker-compose.poc.yml up --build -d
docker compose -f docker-compose.poc.yml exec ollama ollama pull qwen2.5:1.5b
docker compose -f docker-compose.poc.yml exec ollama ollama run qwen2.5:1.5b "Réponds uniquement avec le JSON {\"ready\": true}."
curl --fail http://localhost:8002/health/ready
docker compose -f docker-compose.poc.yml logs --tail=100 backend ollama
docker compose -f docker-compose.poc.yml down
```

Configuration expérimentale par défaut : faster-whisper `base`/CPU/`int8`, Ollama `qwen2.5:1.5b`, fenêtre de scoring de taille 1 et une seule tâche vidéo simultanée. Les modèles, le contexte, le port, le timeout et la taille de batch sont surchargeables par variables d’environnement. Le compose n’impose pas de plafond mémoire : il faut mesurer la consommation réelle.

## Vérification de la phase 1 (25 octobre 2026)

- `docker compose -f docker-compose.poc.yml config --quiet` : réussi.
- L’hôte Docker est maintenant accessible avec autorisation hors sandbox. L’image `ollama/ollama:latest` a été récupérée, et le service `ollama` est `healthy`.
- `qwen2.5:1.5b` a été téléchargé dans le volume persistant Ollama (986 Mo). Une requête de test vers `http://ollama:11434/v1/chat/completions`, depuis un conteneur temporaire sur le réseau Compose, a renvoyé HTTP 200 et le JSON `{"ready": true}`. Le chemin testé n’utilise aucun service IA externe.
- Mesures observées sur l’hôte : 7.176 Gio de RAM totale, 2.0 Gio disponibles après le test, 1.4/4 Gio de swap utilisés, 58 Gio disponibles sur `/`. Ollama utilisait 65.69 Mio avant chargement du modèle et 1.26 Gio après une requête. Ces valeurs décrivent la machine de développement, pas le VPS de 4 Go.
- Diagnostic réseau (25 octobre 2026) : avec Proton VPN actif, `curl -I http://deb.debian.org/debian/dists/trixie/InRelease` renvoie `Empty reply from server`, tandis que la même URL en HTTPS renvoie HTTP 200. Le build échouait donc parce que les sources Debian du Dockerfile utilisaient HTTP, et non parce que le processus APT du Dockerfile manquait de privilèges root. UFW est inactif.
- Correction : les deux étapes APT du `Dockerfile` basculent les sources Debian de `http://deb.debian.org` vers `https://deb.debian.org`. Le build a abouti avec `docker buildx build --network host --load -t shorts-maker-poc-backend:local .`; le réseau hôte est aussi configuré pour les instructions `RUN` du build Compose.
- Le démarrage sur les ports 8000 et 8001 a rencontré des conflits avec d’autres services locaux. Le POC a été lancé sans arrêter ces services en utilisant `SHORTS_MAKER_PORT=8002`.
- Validation du backend : le conteneur est sain, le démarrage Uvicorn et les tâches internes apparaissent dans les logs, et `curl -fsS http://127.0.0.1:8002/health/ready` renvoie `{"status":"ready"}`. FFmpeg 7.1.5, `faster-whisper` 1.2.1 et PyTorch 2.11.0 ont aussi été vérifiés dans le conteneur. Ollama et le test API Qwen décrits plus haut sont également sains.
- Réduction de l’image CPU : l’image initiale faisait 12,4 Go selon `docker image ls`. PyPI installait le wheel CUDA de PyTorch et ses bibliothèques NVIDIA malgré `WHISPER_DEVICE=cpu`. Le Dockerfile installe maintenant les wheels CPU officiels `torch 2.11.0+cpu` et `torchvision 0.26.0+cpu` pour le build par défaut ; l’image mesure 5,85 Go, soit 6,55 Go de moins (environ 53 %). `torch.cuda.is_available()` renvoie `False`. Le chemin de build GPU reste optionnel avec `--build-arg GPU=1` et n’a pas été testé.
- Le build installe FFmpeg dans l’image. Le chargement réel d’un modèle faster-whisper et une génération vidéo de bout en bout restent à vérifier avec un média de test autorisé ; ils ne sont pas couverts par le seul healthcheck.

La phase 1 est validée pour la construction et le démarrage local du backend, Ollama et la réponse Qwen. La transcription effective et le pipeline vidéo restent à valider dans les phases suivantes.

Pour vérifier la configuration sans démarrer les services :

```bash
docker compose -f docker-compose.poc.yml config --quiet
```
