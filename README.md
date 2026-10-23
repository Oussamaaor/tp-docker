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

## Exercice 4 - Sécurité

1. si quelqu'un prend le contrôle de l'appli il a pas les droits root donc il peut faire moins de dégâts. Dans le TP avec l'utilisateur 10001 on pouvait pas écrire dans `/root` (Permission denied)
2. c'est les droits spéciaux de root découpés en petits morceaux (changer le propriétaire d'un fichier, toucher au réseau...). Avec `--cap-drop=ALL` on les enlève tous
3. `--read-only` bloque l'écriture sur tout le système de fichiers du conteneur, `--tmpfs` donne quand même un dossier où on peut écrire, en mémoire, qui disparaît à l'arrêt. Dans le TP `touch /test.txt` donnait Read-only file system mais `touch /tmp/test.txt` marchait
4. parce que beaucoup d'applis ont besoin d'écrire des fichiers temporaires ou du cache, sinon elles plantent
5. non. Le conteneur partage le noyau de l'hôte donc si il y a une faille dans le noyau ou dans Docker ça peut quand même passer. Ça réduit les risques mais c'est pas une VM, il faut aussi garder les images à jour

## Exercice 5 - Diagnostic

1. arrêté il existe encore, on le voit avec `docker ps -a` et on peut lire ses logs ou le relancer. Supprimé il existe plus, on perd ses logs et sa couche inscriptible
2. que le programme s'est mal terminé. 0 tout va bien, autre chose c'est une erreur. Ici 42 c'est le code qu'on a mis avec `exit 42`
3. la sortie standard c'est les messages normaux ("Demarrage de l application"), la sortie d'erreur c'est pour les erreurs ("Erreur simulee"). `docker logs` montre les deux
4. parce que le conteneur s'arrête tout de suite donc on peut pas faire `docker exec` dedans. Les logs restent après l'arrêt et montrent ce qui s'est passé
5. le message peut être pas clair ou même vide (State.Error était vide), alors que le code de sortie dit toujours si ça a marché. Certains codes veulent dire quelque chose, par ex 137 = tué par SIGKILL comme à l'exercice 1

## Exercice 6 - Rotation des logs

1. ils remplissent le disque et quand le disque est plein le serveur et les applis peuvent planter
2. il décide comment et où Docker garde les logs des conteneurs (json-file par défaut, local, ou envoyer vers un autre outil)
3. `docker logs` c'est pour regarder vite les logs d'un conteneur sur la machine, mais ils partent si on supprime le conteneur. Pour le long terme il faut les envoyer dans un outil à part pour les garder et chercher dedans
4. parce qu'un serveur tourne longtemps, sans rotation les fichiers grossissent sans fin. Avec `max-size=1m` et `max-file=2` on garde au max 2 fichiers. Dans le TP l'autre conteneur était en json-file sans options donc sans limite
5. en prod il y a plein de conteneurs et de serveurs, il faut garder les logs assez longtemps et les retrouver vite en cas de panne sans remplir le disque

## Exercice 7 - Inspection d'un conteneur

1. `docker run` crée et démarre un nouveau conteneur à partir d'une image, `docker exec` lance une commande dans un conteneur qui tourne déjà. Dans le TP le `ps` lancé avec exec avait le PID 8 et le processus principal restait le PID 1
2. elle montre les processus qui tournent dans le conteneur, ici `sh` et `sleep 30`
3. pour voir ce qui a été modifié dans le conteneur, par exemple un fichier ajouté ou changé. Ici `A /tmp/diagnostic.txt`
4. le CPU, la mémoire utilisée et la limite, le réseau, le disque et le nombre de processus. Ici 0% de CPU, 836KiB de mémoire et 2 processus
5. parce que ces modifs sont perdues si on recrée le conteneur et personne sait ce qui a été changé. Il faut mettre les changements dans le Dockerfile pour pouvoir refaire pareil
