# Installation rapide de Wazuh avec Docker

Ce guide explique comment installer Wazuh rapidement avec Docker, générer les certificats, changer les mots de passe par défaut et accéder à l’interface web.

---

## 🧰 Avant de commencer — prérequis

- **Docker** et **Docker Compose** installés sur l’hôte
- **Ressources minimales** : 4 vCPU / 8+ Go RAM (plus pour un usage en production)
- **Accèsroot ou sudo** pour exécuter les commandes Docker
- **Ports réseau utilisés** :
  - 1514 (agents → manager)
  - 1515 (enrôlement)
  - 55000 (API)
  - 5601 (Dashboard)
  - 9200 / 9600 (indexeur)

---

## 📦 Récupérer la stack officielle

Clonez le dépôt officiel Wazuh Docker (choisissez la version souhaitée) :

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

Ce dossier contient tout pour un déploiement single-node.

---

## 🔐 Générer les certificats

Wazuh fournit un générateur pour produire les certificats nécessaires (indexer, manager, dashboard). Exécutez la commande fournie une fois :

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Puis ajustez les permissions pour que les conteneurs puissent lire les certificats :

```bash
sudo chown -R 1000:1000 config/wazuh_indexer_ssl_certs/
```

> ⚠️ Ces certificats sont auto-signés. Utilisez une vraie autorité (CA interne ou publique) pour un usage en production.

Remarque : Les certificats auto-signés sont OK pour les tests. Pour la prod, utilisez une CA interne ou publique..

---

## 🚀 Demarrer Wazuh


```bash
docker compose up -d
```

Attendre ~1 minute, puis ouvrir un navigateur :

- https://<IP_DU_HOST_DOCKER>/
- ou https://wazuh.exemple.com/ si un reverse proxy redirige vers le conteneur wazuh.dashboard (port 5601)

Identifiants par défaut :

- Dashboard : admin / SecretPassword
- API : wazuh-wui / MyS3cr37P450r.*-

> ⚠️ Changez ces mot depasse dès le premier connexion.

---

## 🔒 Changer les mots de passe


1. Arrêter la stack

```bash
docker compose down
```

2. Modifier les variables dans docker-compose.yml

Mettre les nouveaux mots de passe.

3. Générer un hash pour l’indexeur

```bash
docker run --rm -ti wazuh/wazuh-indexer:4.11.1 \
	bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
```

Copiez la valeur retournée (le hash).

Éditez `config/wazuh_indexer/internal_users.yml` et remplacez `hash:` pour l'utilisateur concerné par le nouveau hash :

```yaml
admin:
	hash: "<NOUVEAU_HASH_ICI>"
	reserved: true
	backend_roles:
		- "admin"
	description: "Admin user"
```

4. Redémarrez la stack:

```bash
docker compose up -d --force-recreate
```

5. Appliquer la configuration de sécurité:

```bash
docker exec -it single-node-wazuh.indexer-1 bash -lc "
	export INSTALLATION_DIR=/usr/share/wazuh-indexer; \
	export CACERT=$INSTALLATION_DIR/certs/root-ca.pem; \
	export KEY=$INSTALLATION_DIR/certs/admin-key.pem; \
	export CERT=$INSTALLATION_DIR/certs/admin.pem; \
	export JAVA_HOME=/usr/share/wazuh-indexer/jdk; \
	bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
		-cd /usr/share/wazuh-indexer/opensearch-security/ -nhnv \
		-cacert $CACERT -cert $CERT -key $KEY -p 9200 -icl"
```

Reconnectez-vous au dashboard avec le nouveau mot de passe.

---

### 🔑 Modifier le mot de passe de l'API (wazuh-wui)

Éditez `config/wazuh_dashboard/wazuh.yml` et remplacez la valeur `password` :

```yaml
hosts:
	- 1513629884013:
			url: "https://wazuh.manager"
			port: 55000
			username: wazuh-wui
			password: "<NOUVEAU_API_PASSWORD>"
			run_as: false
```

Puis mettre à jour les variables API_USERNAME et API_PASSWORD dans docker-compose.yml.

Recréer les conteneurs :

```bash
docker compose down
docker compose up -d --force-recreate
```

## 🖥️ Enrôler un agent (exemple Linux)

### Depuis le Dashboard

Aller dans **Agents Management → Deploy new agent → Linux**,  
puis copier les commandes proposées selon votre système.

---

### Sur l’agent (exemple Debian/Ubuntu)

Télécharger et installer le paquet :

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.11.1-1_amd64.deb
sudo WAZUH_MANAGER='srv-wazuh.exemple.local' WAZUH_AGENT_NAME='deb' dpkg -i ./wazuh-agent_4.11.1-1_amd64.deb
```

Demarrer l'agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
> 💡 Vérifiez dans le Dashboard que l’agent apparaît avec le statut Active.
