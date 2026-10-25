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
- `docker compose -f docker-compose.poc.yml up --build -d` : non exécuté ; l’accès à `/var/run/docker.sock` renvoie `permission denied`, y compris lors de la tentative hors sandbox.
- Aucun runtime Podman/nerdctl ni binaire hôte `ffmpeg`, `ollama` ou `pytest` n’est disponible. Le backend, le modèle, l’API, Whisper, les logs et les mesures de ressources n’ont donc pas pu être vérifiés manuellement.

La phase est préparée mais reste partiellement validée jusqu’à ce que l’utilisateur courant ait accès à un démon Docker fonctionnel ou exécute ces contrôles sur une machine compatible.

Pour vérifier la configuration sans démarrer les services :

```bash
docker compose -f docker-compose.poc.yml config --quiet
```
