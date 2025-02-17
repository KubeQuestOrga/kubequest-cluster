# Flapi - Cluster K3s - High Availability 

## 🛠 Tech Stack

- **Ansible**: Automatise le déploiement des noeud master / worker sur les serveurs.
- **CI/CD (GitHub Actions)**: Automatise le processus de déploiement des noeuds worker / master.

<br /><br /><br /><br />


## 📚 Documentation 

### Création du Cluster K3s

#### 1. Connexion au VPS / Serveur Dédié
Connectez-vous à votre serveur qui servira de premier nœud master dans votre cluster K3s.

#### 2. Création et Initialisation du Premier Nœud Master
1. Exécutez la commande suivante pour initialiser le cluster K3s et créer le premier nœud master :
```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init --disable traefik --node-taint CriticalAddonsOnly=true:NoExecute --tls-san cluster-k3s.crzcommon.com
```
2. Configurer le nombre de snapshots conservés (checker plus bas l'étape est décrite dans Sauvegarde et Restauration du Cluster K3s)

#### Explications des Flags :
- `server` : Ce flag indique que le nœud sera initialisé comme un serveur dans le cluster K3s. Cela signifie qu'il agira comme un nœud master, gérant l'état du cluster et exécutant des composants du plan de contrôle tels que l'API server, le scheduler, et le controller manager.
- `--cluster-init` : Initialise un nouveau cluster K3s. Ce flag est nécessaire lors de la création du premier serveur dans un cluster K3s pour mettre en place le plan de contrôle et préparer le cluster à joindre d'autres serveurs ou agents.
- `--disable traefik` : Désactive l'installation automatique de Traefik, qui est l'ingress controller par défaut inclus avec K3s. Vous pouvez choisir de désactiver Traefik si vous prévoyez d'utiliser un autre ingress controller ou si vous n'avez pas besoin de cette fonctionnalité.
- `--node-taint CriticalAddonsOnly=true:NoExecute` : Applique un taint au nœud serveur, ce qui empêche les pods qui n'ont pas de tolérance correspondante d'être planifiés sur ce nœud. Ce taint est souvent utilisé pour s'assurer que seuls les pods critiques pour le fonctionnement du cluster soient exécutés sur les nœuds serveur, aidant à garder ces nœuds stables et sécurisés.
- `--tls-san cluster-k3s.crzcommon.com` : Ajoute un Subject Alternative Name (SAN) au certificat TLS généré pour l'API server de Kubernetes. Cela permet d'accéder en toute sécurité à l'API server via le nom de domaine spécifié (cluster-k3s.crzcommon.com dans cet exemple), en plus de son adresse IP. C'est crucial pour les environnements où vous accédez à l'API server de Kubernetes à travers un réseau ou Internet.

<br /><br />

### Joindre de Nouveaux Nœuds au Cluster K3s

#### 1. Obtenir le `K3S_TOKEN` pour Joindre le Cluster
```bash
# Sur le nœud master
sudo chmod 644 /var/lib/rancher/k3s/server/ && sudo cat /var/lib/rancher/k3s/server/token
```

#### 2. A faire avant de rejoindre le Cluster K3s
1. `ATTENTION` : Avant d'essayer de rejoindre le cluster K3S, il faut voir si le nom de domaine est bien résolu depuis la machine qui veut rejoindre.
```bash
# Erreur :
debian@vps-7b992047:~$ curl -k https://cluster-k3s.crzcommon.com:6443
curl: (6) Could not resolve host: cluster-k3s.crzcommon.com

# Valide :
debian@vps-370a6b62:~$ curl -k https://cluster-k3s.crzcommon.com:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```
Si c'est en erreur, il faudrai ajouté dans /etc/hosts pour résoudre le domaine dans certains cas (ajouter l'ip du loadbalancer pour le domaine) :
```bash
sudo nano /etc/hosts

# Ajouter cette ligne au fichier
149.202.48.239 cluster-k3s.crzcommon.com
```
2. `ATTENTION` : Avant de rejoindre le cluster K3S, il faut absolument le SUPPRIMER du cluster réellement avant qu'il puisse rejoindre à nouveau,
c'est possible qu'ont ce retrouve avec des problèmes pour rejoindre ! Donc il vaut mieux le supprimer du cluster si l'ip était déjà connu auparavant du cluster. <br />
Command : `sudo kubectl delete node <name-node>` (depuis un autre noeud master actif)

#### 3. Joindre un Nœud Master au Cluster K3s 
1. Commande pour rejoindre le cluster K3s en tant que noeud MASTER :
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=xxx sh -s - server --server https://cluster-k3s.crzcommon.com:6443 --disable traefik --node-taint CriticalAddonsOnly=true:NoExecute --tls-san cluster-k3s.crzcommon.com
```
2. Configurer le nombre de snapshots conservés (checker plus bas l'étape est décrite dans Sauvegarde et Restauration du Cluster K3s)

#### 4. Joindre un Nœud Worker au Cluster K3s
1. Commande pour rejoindre le cluster K3s en tant que noeud WORKER :
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=xxx sh -s - agent --server https://cluster-k3s.crzcommon.com:6443
```

#### 5. Après avoir rejoindre SI JAMAIS il y a une erreur
`ATTENTION`: Si vous avez l'erreur ci-dessous, il est possible que le noeud vas quand même rejoindre patienter jusqu'a 5min.
```bash
Job for k3s.service failed because the control process exited with error code.
See "systemctl status k3s.service" and "journalctl -xeu k3s.service" for details.
```

#### 6. Configuration Système et Conseils
- **Désactiver le Swap (vps pas possible)** : Important pour les performances et la stabilité de Kubernetes.
- **Nombre Impair de Nœuds Master** : Nécessaire pour maintenir le quorum en cas de panne.
- **Noms d'Hôtes Uniques** : Chaque nœud dans le cluster doit avoir un nom d'hôte unique.
- **Utiliser des Disques SSD pour les Nœuds Master** : Recommandé pour de meilleures performances et une latence réduite.
- **Désactiver le parefeu (Ubuntu/Debian)** : `ufw disable`
- **Mettre à jour le système**: `sudo apt update && sudo apt upgrade -y`

<br /><br />

### Mise à jour de la version Kubernetes sur les noeuds
1. Se connecter sur un noeud.
2. Run command : 
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable sh -
```
3. Check si la monté de version c'est bien fait :
```bash
sudo kubectl get node -o wide
```

<br /><br />

### Haute Disponibilité du Cluster K3s

#### Un cluster HA K3s avec etcd intégré est composé de :
- Trois nœuds de serveur ou plus qui serviront l'API Kubernetes et exécuteront d'autres services de plan de contrôle, ainsi qu'hébergeront la banque de données etcd intégrée. 
- Facultatif : zéro ou plusieurs nœuds d'agent désignés pour exécuter vos applications et services
- Facultatif : une adresse d'enregistrement fixe (load balancer) pour que les nœuds d'agent / worker s'inscrivent auprès du cluster (Voir le projet : https://github.com/CrzGames/Crzgames_LoadBalancer_External)

<br /><br />

### Get KUBECONFIG :
1. Connect to the NODE MASTER in the Cluster K3S.
2. Run command, copy kubeconfig : 
```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```
3. Replace "127.0.0.1:6443" to " https://cluster-k3s.crzcommon.com:6443".
4. Encode BASE64 and add in SECRETS VARIABLE for CI / CD, run command :
```bash
sudo base64 /etc/rancher/k3s/k3s.yaml > k3s_base64.txt 
```

<br /><br />

## Sauvegarde et Restauration du Cluster K3s
### Par défault
- K3s crée des `snapshots etcd automatiques`.
- K3s crée les snapshots etcd automatique à `00h00` et `12h00`
- K3s `conserve` les `5 derniers snapshots`.
- K3s conserve les snapshots dans : `/var/lib/rancher/k3s/server/db/snapshots`.
- Lorsque K3s est restauré à partir d'une sauvegarde, l'ancien répertoire de données (`/var/lib/rancher/k3s/server/db/snapshots`) est déplacé vers `/var/lib/rancher/k3s/server/db/etcd-old/`. K3s tente ensuite de restaurer l'instantané en créant un nouveau répertoire de données, puis en démarrant etcd avec un nouveau cluster K3s avec un membre etcd.

### Configurer le nombre de snapshots conservés (à faire sur chaque noeud master)
```bash
# Ajoutez l'option '--etcd-snapshot-retention=10' à la ligne ExecStart :
sudo nano /etc/systemd/system/k3s.service
# Recharger le daemon systemd pour appliquer les modifications :
sudo systemctl daemon-reload
# Arreter et démarrer à nouveau K3s :
sudo systemctl stop k3s
sudo systemctl start k3s
```

### Création d'un Snapshot etcd Manuel
```bash
sudo k3s etcd-snapshot save
# save is : /var/lib/rancher/k3s/server/db/snapshots
```

### Liste des Snapshots
```bash
sudo k3s etcd-snapshot ls
```

### Restaurer un Cluster K3s
1. Sur le premier nœud (Server1), arrêter et démarrer K3s avec les options de réinitialisation du cluster :
```bash
sudo systemctl stop k3s
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/mysnapshot
```
2. Sur les autres nœuds (Server2, Server3), arrêtez K3s et supprimez le répertoire de données :
```bash
sudo systemctl stop k3s
sudo rm -rf /var/lib/rancher/k3s/server/db/
```
3. Sur le premier nœud (Sever1), redémarrez K3s :
```bash
sudo systemctl start k3s
```
4. Sur les autres nœuds (Server2, Server3), redémarrez K3s pour rejoindre le cluster restauré :
```bash
sudo systemctl start k3s
```

<br /><br /><br /><br />


## 🚀 Production
### ⚙️➡️ Processus de distribution automatique (CI / CD)
#### Configuration Initiale pour un Nouveau Projet

1. **Créer le Cluster K3S**: Il faudra au préalable avoir crée manuellement le cluster K3S, voir tout en haut du fichier 'README.md' pour plus de détails.

2. **Configuration des secrets GitHub**:

   Pour la configuration initiale, assurez-vous d'ajouter les secrets suivants dans votre dépôt GitHub :

   - `CLUSTER_SSH_PRIVATE_KEY`: Votre clé privée SSH. Cette clé doit correspondre à la clé publique déjà installée sur les serveurs cibles.
   - `CLUSTER_SSH_HOSTS`: Les adresses IP des serveurs cibles, séparées par des espaces. <br />
   Exemple : `192.0.2.1 198.51.100.2`. Ces adresses seront utilisées pour préparer le fichier `known_hosts`, facilitant ainsi des connexions SSH sécurisées.
   - `CLUSTER_K3S_TOKEN`: Token pour s'authentifier au près du cluster K3S, on peut le récupérer `via` un `noeud master` : <br />
    ```bash
    sudo chmod 644 /var/lib/rancher/k3s/server/ && sudo cat /var/lib/rancher/k3s/server/token
    ```

3. **Ajout de la clé publique sur les serveurs cibles**:

   Avant de pouvoir déployer automatiquement via CI/CD, la clé publique associée à `CLUSTER_SSH_PRIVATE_KEY` doit être ajoutée au fichier `~/.ssh/authorized_keys` sur chaque serveur cible. Cette étape garantit que GitHub Actions peut se connecter aux serveurs via SSH sans intervention manuelle.
