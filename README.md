# TP Docker avancé

## Exercice 1 - Cycle de vie des processus

1. le processus principal c'est la commande lancée au démarrage du conteneur (ici `sleep 300`). Le conteneur vit tant que ce processus tourne
2. `docker stop` envoie SIGTERM, attend un peu et envoie SIGKILL si ça s'arrête pas. `docker kill` envoie direct le signal (SIGKILL par défaut ou celui qu'on met avec `--signal`)
3. parce que le conteneur existe juste pour faire tourner ce processus. Quand il est fini il y a plus rien à faire donc le conteneur s'arrête
4. SIGTERM demande à l'appli de s'arrêter, elle peut finir ce qu'elle fait avant. SIGKILL la tue direct. Dans le TP `docker stop` a donné le code 137 (tué par SIGKILL) parce que `sleep` a ignoré le SIGTERM, et avec `docker kill --signal=TERM` le conteneur est resté en running
5. pour qu'elle s'arrête proprement (finir les requêtes, sauvegarder, fermer les connexions) et pas être coupée au milieu

## Exercice 2 - Couches de l'image

1. les couches de l'image sont en lecture seule, c'est le modèle. Quand on lance un conteneur Docker met une couche inscriptible par dessus et toutes les modifs vont dedans
2. quand on modifie un fichier de l'image, Docker le copie d'abord dans la couche du conteneur et c'est la copie qui change. L'image reste pareille et on a pas besoin de copier toute l'image pour chaque conteneur
3. parce que les modifs vont dans la couche du conteneur et pas dans l'image. Plusieurs conteneurs peuvent utiliser la même image sans se gêner
4. elle montre ce qui a changé dans le conteneur par rapport à l'image : A ajouté, C modifié, D supprimé. Dans le TP on avait `A /tmp/test.txt`, `A /opt/test` et `C /etc/alpine-release`
5. parce que la couche inscriptible appartient au conteneur, elle part avec lui. Le nouveau conteneur affichait juste `3.22.7` sans la ligne ajoutée. Pour garder des données il faut un volume

## Exercice 3 - Multi-stage build

1. chaque `FROM` commence une nouvelle étape. Ça permet d'avoir une étape pour compiler et une autre pour l'image finale
2. l'étape `builder` a l'image golang avec le compilateur, elle sert juste à faire l'exécutable. L'étape finale part de `scratch` (vide) et on met juste l'exécutable dedans
3. parce qu'avec `COPY --from=builder` on copie seulement `/app`. Le reste de l'étape builder est pas gardé dans l'image finale
4. l'image est beaucoup plus légère (12,8 Mo ici), plus rapide à télécharger et à envoyer sur Docker Hub, et il y a moins de choses dedans donc moins de failles
5. il y a pas de shell ni d'outils donc on peut pas faire `docker exec` pour chercher un problème. Il faut compiler en statique (`CGO_ENABLED=0`) et le processus tourne en root par défaut (vu avec `docker top`) alors qu'il vaut mieux éviter root
