# Runtime local du POC

Le profil `docker-compose.poc.yml` lance seulement FastAPI et Ollama. Le backend est accessible sur `http://localhost:8000`, tandis qu’Ollama reste privé au réseau Docker. Les données Ollama, les imports, les exports et le cache Whisper sont conservés dans des volumes nommés.

```bash
docker compose -f docker-compose.poc.yml up --build -d
docker compose -f docker-compose.poc.yml exec ollama ollama pull qwen2.5:1.5b
docker compose -f docker-compose.poc.yml exec ollama ollama run qwen2.5:1.5b "Réponds uniquement avec le JSON {\"ready\": true}."
curl --fail http://localhost:8000/health/ready
docker compose -f docker-compose.poc.yml logs --tail=100 backend ollama
docker compose -f docker-compose.poc.yml down
```

Configuration expérimentale par défaut : faster-whisper `base`/CPU/`int8`, Ollama `qwen2.5:1.5b`, fenêtre de scoring de taille 1 et une seule tâche vidéo simultanée. Les modèles, le contexte, le port, le timeout et la taille de batch sont surchargeables par variables d’environnement. Le compose n’impose pas de plafond mémoire : il faut mesurer la consommation réelle.

## Vérification de la phase 1 (25 octobre 2026)

- `docker compose -f docker-compose.poc.yml config --quiet` : réussi.
- L’hôte Docker est maintenant accessible avec autorisation hors sandbox. L’image `ollama/ollama:latest` a été récupérée, et le service `ollama` est `healthy`.
- `qwen2.5:1.5b` a été téléchargé dans le volume persistant Ollama (986 Mo). Une requête de test vers `http://ollama:11434/v1/chat/completions`, depuis un conteneur temporaire sur le réseau Compose, a renvoyé HTTP 200 et le JSON `{"ready": true}`. Le chemin testé n’utilise aucun service IA externe.
- Mesures observées sur l’hôte : 7.176 Gio de RAM totale, 2.0 Gio disponibles après le test, 1.4/4 Gio de swap utilisés, 58 Gio disponibles sur `/`. Ollama utilisait 65.69 Mio avant chargement du modèle et 1.26 Gio après une requête. Ces valeurs décrivent la machine de développement, pas le VPS de 4 Go.
- `docker compose -f docker-compose.poc.yml up --build -d` : le pull Ollama a réussi, mais le build backend s’est arrêté à `apt-get update`. Le réseau ne joint pas `deb.debian.org` sur le port 80 depuis le build (`Connection failed`) ; les contrôles HTTPS depuis l’hôte échouent également (`SSL_ERROR_SYSCALL`).
- Comme le build ne termine pas, le backend, ffmpeg dans l’image, le healthcheck FastAPI et le chargement Whisper n’ont pas encore été vérifiés.

La phase reste partiellement validée. Il faut rétablir l’accès réseau du build Docker vers les dépôts Debian et les registries de dépendances, ou fournir un miroir/proxy accessible, puis reprendre le build et les contrôles backend/Whisper.

Pour vérifier la configuration sans démarrer les services :

```bash
docker compose -f docker-compose.poc.yml config --quiet
```
