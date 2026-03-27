------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
*La perte d'un pod Flask seul n'entraîne pas de perte de données car les données ne sont pas stockées dans le système de fichiers éphémère du conteneur mais dans le PVC pra-data. En revanche, la perte de certains composants entraîne bien une perte de données ou un risque majeur de perte:

- Le PVC pra-data: volume principal qui contient le fichier SQLite app.db. Sa suppression provoque la perte immédiate de la base en production.
- Le PV/support physique associé au PVC pra-data: même effet que ci-dessus car le contenu réel du PVC disparaît.
- Le PVC pra-backup: sa perte ne supprime pas forcément la base de production immédiatement, mais elle supprime la capacité de restauration. En cas d'incident ultérieur sur pra-data, les données seraient alors perdues définitivement.
- Le stockage local du cluster K3d/du noeud hôte: dans cet atelier, les volumes restent locaux. Une panne de la machine hôte, une corruption disque ou une suppression du cluster peut donc impacter à la fois pra-data et pra-backup.

Les composants critiques pour les données sont donc principalement pra-data (données en production) et pra-backup (capacité de reprise), ainsi que l'infrastructure de stockage sous-jacente.*

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
*Au moment exact où pra-data est supprimé, les données de production sont bien perdues. En revanche, nous ne les avons pas perdues définitivement, car une copie existait déjà dans pra-backup grâce au CronJob sqlite-backup qui sauvegarde la base toutes les minutes.

Le mécanisme est donc le suivant:

1. La base active est stockée dans pra-data.
2. Des sauvegardes régulières sont copiées depuis pra-data vers pra-backup.
3. Lors du sinistre, pra-data est supprimé: l'application ne retrouve plus sa base de données.
4. On recrée un volume pra-data vide.
5. On exécute ensuite le job de restauration qui recopie un fichier .db depuis pra-backup vers pra-data.

Donc, la raison pour laquelle les données réapparaissent après l'incident n'est pas que Kubernetes protège automatiquement le contenu du PVC supprimé mais bien que nous avions anticipé le sinistre avec une sauvegarde séparée.

Il y a eu une perte de la base de production, mais pas une perte définitive des données car elles ont été restaurées depuis le volume de sauvegarde.*

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
*Les valeurs de RTO et RPO dépendent ici du scénario observé.

1) En cas de crash du pod (PCA)
- RTO: très faible, généralement de quelques secondes, le temps que Kubernetes recrée le pod.
- RPO: 0 car aucune donnée n'est perdue : la base reste stockée sur le PVC pra-data.

2) En cas de perte de pra-data (PRA)
- RTO : non nul car une intervention manuelle est nécessaire : arrêt du déploiement, suspension du CronJob, recréation de l'infrastructure, exécution du job de restauration, vérification puis reprise des sauvegardes. Dans cet atelier, on peut estimer un RTO de quelques minutes, selon la rapidité d'exécution de l'opérateur.
- RPO: jusqu'à 1 minute car la sauvegarde est effectuée toutes les minutes. On peut donc perdre au maximum les écritures réalisées depuis le dernier backup.

En résumé:
- Scénario PCA (perte du pod) => RTO court, RPO = 0.
- Scénario PRA (perte de pra-data) => RTO de quelques minutes, RPO ≈ 1 minute maximum.*

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
*Cet atelier est très utile pour comprendre les principes mais il ne peut pas être utilisé tel quel en production car il manque plusieurs garanties indispensables:

- Pas de redondance géographique ni de vrai site de secours: les données et les sauvegardes restent dans le même environnement Kubernetes local.
- Backups non externalisés: pra-data et pra-backup peuvent être perdus ensemble si l'hôte, le cluster ou le stockage local tombent.
- Base SQLite non adaptée à une production distribuée: SQLite est simple et pratique pour un atelier mais limitée pour une application critique multi-utilisateurs.
- Restauration manuelle: il faut déclencher plusieurs commandes à la main. En production, il faut un runbook industrialisé, testé et si possible automatisé.
- Pas de supervision complète: il manque des alertes, des métriques, des journaux centralisés et des contrôles de succès des sauvegardes/restaurations.
- Pas de tests réguliers de PRA: une sauvegarde n'a de valeur que si la restauration est testée fréquemment.
- Pas de sécurité avancée: pas de chiffrement des sauvegardes, pas de gestion d'accès fine, pas de politique de rétention, pas d'immuabilité.
- Pas d'objectifs contractuels garantis: les RTO/RPO sont observés dans un atelier mais pas garantis par une architecture robuste.

Ce qui manque principalement: un stockage répliqué, des sauvegardes externalisées, une architecture multi-zones/multi-sites, une base de données adaptée à la production, de la supervision, de l'automatisation et des tests réguliers de reprise.*
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
*Une architecture plus robuste pourrait être la suivante:

1) Couche applicative
- Plusieurs réplicas de l'application Flask derrière un Service Kubernetes.
- Répartition de charge et éventuellement un Ingress pour l'exposition applicative.
- Déploiement sur plusieurs nœuds et idéalement sur plusieurs zones de disponibilité.

2) Couche base de données
- Remplacer SQLite par une base de données de production, par exemple PostgreSQL.
- Mettre en place une **réplication** (primaire + réplique) ou utiliser un service managé.
- Prévoir une haute disponibilité de la base avec bascule automatique.

3) Sauvegarde / PRA
- Sauvegardes régulières vers un stockage externe (par exemple un bucket objet).
- Politique de rétention claire : journalière, hebdomadaire, mensuelle.
- Possibilité de restauration à un instant choisi et non uniquement depuis le dernier backup.
- Vérification automatique de l'intégrité des sauvegardes.

4) PCA / infrastructure
- Cluster Kubernetes haute disponibilité.
- Stockage persistant répliqué.
- Déploiement sur deux sites ou deux régions selon les besoins métiers.

5) Supervision et sécurité
- Supervision des pods, volumes, jobs de backup et temps de restauration.
- Alertes en cas d'échec de sauvegarde ou d'absence de backup récent.
- Chiffrement des données et des sauvegardes.
- Contrôle d'accès fort et journalisation centralisée.

Conclusion: une architecture robuste repose sur la combinaison de haute disponibilité (PCA), sauvegardes externalisées, procédures de restauration testées, stockage répliqué et base de données adaptée à la production.*

---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

![alt text](atelier1screen.png)

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

*..Décrir ici votre procédure de restauration (votre runbook)..*  
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

