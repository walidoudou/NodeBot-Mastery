# Chapitre 5 - Création d'un Bot Twitch

Dans ce chapitre, nous allons explorer la création de bots pour la plateforme de streaming Twitch. Les bots Twitch peuvent améliorer l'expérience de streaming en automatisant des tâches, en interagissant avec les spectateurs et en fournissant des fonctionnalités supplémentaires au streamer.

## Introduction aux bots Twitch

Les bots Twitch sont des programmes automatisés qui peuvent interagir avec le chat Twitch et effectuer diverses actions comme :

- Répondre aux commandes des spectateurs
- Modérer le chat en supprimant les messages indésirables
- Suivre les statistiques du stream (nouveaux abonnés, followers, etc.)
- Organiser des jeux et concours
- Gérer des systèmes de points de fidélité personnalisés
- Jouer de la musique ou afficher des informations à l'écran

## Préparation et configuration

### Obtenir les identifiants Twitch

Pour créer un bot Twitch, vous devez d'abord enregistrer une application sur le portail des développeurs Twitch :

1. Connectez-vous au [Portail des développeurs Twitch](https://dev.twitch.tv/)
2. Accédez à la [Console](https://dev.twitch.tv/console)
3. Cliquez sur "Enregistrer une application"
4. Remplissez les informations requises :
   - Nom de l'application (le nom de votre bot)
   - URL de redirection OAuth (utilisez `http://localhost` pour le développement)
   - Catégorie (choisissez "Chat Bot")
5. Cliquez sur "Créer"
6. Notez votre Client ID
7. Générez et notez votre Client Secret

### Configuration du projet

Créons la structure de base pour notre bot Twitch :

1. Créez un nouveau dossier pour votre projet :

```bash
mkdir mon-bot-twitch
cd mon-bot-twitch
```

2. Initialisez un projet Node.js :

```bash
npm init -y
```

3. Installez les dépendances nécessaires :

```bash
npm install tmi.js dotenv
```

4. Créez un fichier `.env` pour stocker vos identifiants :

```
TWITCH_USERNAME=votre_nom_utilisateur_bot
TWITCH_OAUTH_TOKEN=oauth:votre_token_oauth
TWITCH_CLIENT_ID=votre_client_id
TWITCH_CLIENT_SECRET=votre_client_secret
TWITCH_CHANNELS=canal1,canal2
```

5. Pour obtenir votre token OAuth, vous pouvez utiliser le [générateur de token Twitch](https://twitchapps.com/tmi/) ou créer votre propre flux OAuth.

6. Créez la structure de dossiers suivante :

```
mon-bot-twitch/
│
├── node_modules/
├── src/
│   ├── commands/      # Commandes du bot
│   ├── utils/         # Fonctions utilitaires
│   ├── events/        # Gestionnaires d'événements
│   └── index.js       # Point d'entrée principal
│
├── .env               # Variables d'environnement
├── .gitignore
└── package.json
```

7. Créez un fichier `.gitignore` :

```
node_modules/
.env
```

## Premier bot Twitch fonctionnel

Commençons par créer un bot Twitch simple qui répond à des commandes de base :

### 1. Fichier principal (src/index.js)

```javascript
// src/index.js
require("dotenv").config();
const tmi = require("tmi.js");
const fs = require("fs");
const path = require("path");

// Configuration du client Twitch
const client = new tmi.Client({
  options: { debug: true },
  identity: {
    username: process.env.TWITCH_USERNAME,
    password: process.env.TWITCH_OAUTH_TOKEN,
  },
  channels: process.env.TWITCH_CHANNELS.split(","),
});

// Collection pour stocker les commandes
const commands = new Map();

// Chargement des commandes
const commandsPath = path.join(__dirname, "commands");
const commandFiles = fs
  .readdirSync(commandsPath)
  .filter((file) => file.endsWith(".js"));

for (const file of commandFiles) {
  const filePath = path.join(commandsPath, file);
  const command = require(filePath);

  if ("name" in command && "execute" in command) {
    commands.set(command.name, command);
    console.log(`✓ Commande chargée : ${command.name}`);
  } else {
    console.warn(
      `⚠ La commande ${file} manque de propriétés 'name' ou 'execute'`
    );
  }
}

// Événement : Connexion réussie
client.on("connected", (address, port) => {
  console.log(`✅ Bot connecté à ${address}:${port}`);
});

// Événement : Message reçu
client.on("message", async (channel, tags, message, self) => {
  // Ignorer les messages du bot lui-même
  if (self) return;

  // Préfixe des commandes
  const prefix = "!";

  // Vérifier si le message commence par le préfixe
  if (!message.startsWith(prefix)) return;

  // Extraire le nom de la commande et les arguments
  const args = message.slice(prefix.length).trim().split(/\s+/);
  const commandName = args.shift().toLowerCase();

  // Vérifier si la commande existe
  if (!commands.has(commandName)) return;

  // Exécuter la commande
  try {
    await commands.get(commandName).execute(client, channel, tags, args);
  } catch (error) {
    console.error(
      `Erreur lors de l'exécution de la commande ${commandName}:`,
      error
    );
    client.say(
      channel,
      `Une erreur s'est produite lors de l'exécution de cette commande.`
    );
  }
});

// Connexion à Twitch
client.connect().catch(console.error);
```

### 2. Commandes de base (src/commands/)

#### Commande Ping (src/commands/ping.js)

```javascript
// src/commands/ping.js
module.exports = {
  name: "ping",
  description: "Répond avec pong et affiche la latence",
  execute: async (client, channel, tags, args) => {
    const startTime = Date.now();
    await client.ping();
    const latency = Date.now() - startTime;

    client.say(channel, `Pong ! 🏓 Latence : ${latency}ms`);
  },
};
```

#### Commande d'aide (src/commands/aide.js)

```javascript
// src/commands/aide.js
const fs = require("fs");
const path = require("path");

module.exports = {
  name: "aide",
  description: "Affiche la liste des commandes disponibles",
  execute: async (client, channel, tags, args) => {
    // Si un nom de commande spécifique est fourni
    if (args.length > 0) {
      const commandName = args[0].toLowerCase();
      const commandsPath = path.join(__dirname);
      const commandFiles = fs
        .readdirSync(commandsPath)
        .filter((file) => file.endsWith(".js"));

      for (const file of commandFiles) {
        const command = require(path.join(commandsPath, file));
        if (command.name === commandName) {
          return client.say(
            channel,
            `!${command.name}: ${command.description}`
          );
        }
      }

      return client.say(channel, `Commande "${commandName}" non trouvée.`);
    }

    // Sinon afficher la liste des commandes
    const commandsPath = path.join(__dirname);
    const commandFiles = fs
      .readdirSync(commandsPath)
      .filter((file) => file.endsWith(".js"));

    let helpMessage = "Commandes disponibles : ";

    for (const file of commandFiles) {
      const command = require(path.join(commandsPath, file));
      helpMessage += `!${command.name} `;
    }

    helpMessage +=
      '- Utilisez "!aide [nom_commande]" pour plus d\'informations sur une commande spécifique.';

    client.say(channel, helpMessage);
  },
};
```

#### Commande pour les suiveurs (src/commands/uptime.js)

```javascript
// src/commands/uptime.js
module.exports = {
  name: "uptime",
  description: "Affiche depuis combien de temps le stream est en ligne",
  execute: async (client, channel, tags, args) => {
    // Le nom du canal sans le # au début
    const channelName = channel.slice(1);

    try {
      // Requête à l'API Twitch pour obtenir les informations du stream
      // Note: Ceci est un exemple simplifié, vous devriez idéalement utiliser
      // la bibliothèque API Twitch pour Node.js ou implémenter la gestion du token

      client.say(
        channel,
        `Le stream est en ligne depuis 2 heures 30 minutes. (Simulation)`
      );

      // Dans une implémentation réelle, vous feriez une requête à l'API Twitch:
      /*
            const response = await fetch(`https://api.twitch.tv/helix/streams?user_login=${channelName}`, {
                headers: {
                    'Client-ID': process.env.TWITCH_CLIENT_ID,
                    'Authorization': `Bearer ${accessToken}`
                }
            });
            
            const data = await response.json();
            
            if (data.data.length === 0) {
                return client.say(channel, `${channelName} n'est pas en ligne actuellement.`);
            }
            
            const startTime = new Date(data.data[0].started_at);
            const uptime = calculateUptime(startTime);
            
            client.say(channel, `${channelName} est en ligne depuis ${uptime}.`);
            */
    } catch (error) {
      console.error(
        "Erreur lors de la récupération des informations du stream:",
        error
      );
      client.say(
        channel,
        `Impossible de récupérer les informations du stream.`
      );
    }
  },
};

// Fonction pour calculer le temps d'uptime
function calculateUptime(startTime) {
  const currentTime = new Date();
  const diffInMs = currentTime - startTime;

  const hours = Math.floor(diffInMs / (1000 * 60 * 60));
  const minutes = Math.floor((diffInMs % (1000 * 60 * 60)) / (1000 * 60));

  let uptimeStr = "";

  if (hours > 0) {
    uptimeStr += `${hours} heure${hours > 1 ? "s" : ""} `;
  }

  uptimeStr += `${minutes} minute${minutes > 1 ? "s" : ""}`;

  return uptimeStr;
}
```

#### Commande personnalisée (src/commands/social.js)

```javascript
// src/commands/social.js
module.exports = {
  name: "social",
  description: "Affiche les liens vers les réseaux sociaux",
  execute: async (client, channel, tags, args) => {
    // Message avec les réseaux sociaux du streamer
    const socialMessage =
      "Suivez-moi sur Twitter: https://twitter.com/votre_pseudo • Instagram: https://instagram.com/votre_pseudo • Discord: https://discord.gg/votre_serveur";

    client.say(channel, socialMessage);
  },
};
```

#### Commande de sondage simple (src/commands/sondage.js)

```javascript
// src/commands/sondage.js
module.exports = {
  name: "sondage",
  description: "Crée un sondage simple",
  cooldown: 60, // Cooldown en secondes
  execute: async (client, channel, tags, args) => {
    // Vérifier si l'utilisateur est modérateur ou streamer
    const isMod = tags.mod || tags.username === channel.slice(1);

    if (!isMod) {
      return client.say(
        channel,
        `@${tags.username}, seuls les modérateurs peuvent créer des sondages.`
      );
    }

    // Vérifier si la question et les options sont fournies
    if (args.length < 3) {
      return client.say(
        channel,
        `@${tags.username}, utilisation : !sondage "Question" "Option 1" "Option 2" [Option 3] ...`
      );
    }

    // Reconstruire la commande pour analyser correctement les guillemets
    const fullCommand = args.join(" ");
    const matches = fullCommand.match(/"([^"]*)"/g);

    if (!matches || matches.length < 3) {
      return client.say(
        channel,
        `@${tags.username}, format incorrect. Utilisez : !sondage "Question" "Option 1" "Option 2" [Option 3] ...`
      );
    }

    // Extraire la question et les options
    const question = matches[0].replace(/"/g, "");
    const options = matches.slice(1).map((opt) => opt.replace(/"/g, ""));

    // Limiter le nombre d'options (pour cet exemple, maximum 4)
    const limitedOptions = options.slice(0, 4);

    // Créer le message de sondage
    let pollMessage = `📊 SONDAGE : ${question}\n`;

    limitedOptions.forEach((option, index) => {
      pollMessage += `${index + 1}) ${option} `;
    });

    pollMessage += `\nVotez en tapant le numéro correspondant dans le chat !`;

    // Envoyer le sondage
    client.say(channel, pollMessage);

    // Dans une implémentation réelle, vous pourriez maintenant:
    // 1. Configurer un timer pour la durée du sondage
    // 2. Écouter les messages du chat pour compter les votes
    // 3. Afficher les résultats à la fin du sondage
  },
};
```

## Système de points et de fidélité

Les systèmes de points sont très populaires sur Twitch. Créons un système simple de points (ou "monnaie" du stream) :

### 1. Utilitaire pour gérer les points (src/utils/pointsSystem.js)

```javascript
// src/utils/pointsSystem.js
const fs = require("fs").promises;
const path = require("path");

// Chemin vers le fichier de données
const DATA_PATH = path.join(__dirname, "..", "..", "data", "points.json");

// Classe de gestion des points
class PointsSystem {
  constructor() {
    this.points = {};
    this.loaded = false;

    // Points gagnés par défaut toutes les 10 minutes
    this.defaultPointsPerInterval = 10;

    // Intervalles d'attribution de points actifs par canal
    this.activeIntervals = {};
  }

  // Charger les données depuis le fichier
  async load() {
    try {
      const data = await fs.readFile(DATA_PATH, "utf8");
      this.points = JSON.parse(data);
    } catch (error) {
      // Si le fichier n'existe pas ou est invalide, créer un objet vide
      this.points = {};

      // Créer le répertoire de données s'il n'existe pas
      try {
        await fs.mkdir(path.dirname(DATA_PATH), { recursive: true });
      } catch (err) {
        // Ignorer si le dossier existe déjà
      }
    }

    this.loaded = true;
    console.log("Système de points chargé");
  }

  // Sauvegarder les données dans le fichier
  async save() {
    if (!this.loaded) await this.load();

    try {
      await fs.writeFile(
        DATA_PATH,
        JSON.stringify(this.points, null, 2),
        "utf8"
      );
    } catch (error) {
      console.error("Erreur lors de la sauvegarde des points:", error);
    }
  }

  // Ajouter des points à un utilisateur
  async addPoints(channel, username, amount) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal (sans le #)
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Initialiser la structure si nécessaire
    if (!this.points[normalizedChannel]) {
      this.points[normalizedChannel] = {};
    }

    if (!this.points[normalizedChannel][username]) {
      this.points[normalizedChannel][username] = 0;
    }

    // Ajouter les points
    this.points[normalizedChannel][username] += amount;

    // Sauvegarder les changements
    await this.save();

    return this.points[normalizedChannel][username];
  }

  // Retirer des points à un utilisateur
  async removePoints(channel, username, amount) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si l'utilisateur a des points
    if (
      !this.points[normalizedChannel] ||
      !this.points[normalizedChannel][username]
    ) {
      return 0;
    }

    // Retirer les points (minimum 0)
    this.points[normalizedChannel][username] = Math.max(
      0,
      this.points[normalizedChannel][username] - amount
    );

    // Sauvegarder les changements
    await this.save();

    return this.points[normalizedChannel][username];
  }

  // Obtenir les points d'un utilisateur
  async getPoints(channel, username) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si l'utilisateur a des points
    if (
      !this.points[normalizedChannel] ||
      !this.points[normalizedChannel][username]
    ) {
      return 0;
    }

    return this.points[normalizedChannel][username];
  }

  // Obtenir le classement des utilisateurs
  async getLeaderboard(channel, limit = 10) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si le canal a des données
    if (!this.points[normalizedChannel]) {
      return [];
    }

    // Créer un tableau d'objets {username, points}
    const leaderboard = Object.entries(this.points[normalizedChannel])
      .map(([username, points]) => ({ username, points }))
      .sort((a, b) => b.points - a.points)
      .slice(0, limit);

    return leaderboard;
  }

  // Démarrer l'attribution automatique de points
  startAutoPointsInterval(client, channel, intervalMinutes = 10) {
    // Arrêter l'intervalle existant si présent
    this.stopAutoPointsInterval(channel);

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    console.log(
      `Démarrage de l'attribution automatique de points pour ${normalizedChannel} toutes les ${intervalMinutes} minutes`
    );

    // Créer un nouvel intervalle
    const intervalId = setInterval(async () => {
      try {
        // Obtenir la liste des chatters actifs (ceci est une simulation)
        // Dans une implémentation réelle, vous utiliseriez l'API Twitch
        const activeChatters = await this.getActiveChatters(client, channel);

        // Attribuer des points à chaque chatter
        for (const username of activeChatters) {
          await this.addPoints(
            normalizedChannel,
            username,
            this.defaultPointsPerInterval
          );
        }

        console.log(
          `Points attribués à ${activeChatters.length} chatters sur ${normalizedChannel}`
        );
      } catch (error) {
        console.error(
          `Erreur lors de l'attribution automatique de points:`,
          error
        );
      }
    }, intervalMinutes * 60 * 1000);

    // Stocker l'ID de l'intervalle
    this.activeIntervals[normalizedChannel] = intervalId;
  }

  // Arrêter l'attribution automatique de points
  stopAutoPointsInterval(channel) {
    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Arrêter l'intervalle s'il existe
    if (this.activeIntervals[normalizedChannel]) {
      clearInterval(this.activeIntervals[normalizedChannel]);
      delete this.activeIntervals[normalizedChannel];

      console.log(
        `Attribution automatique de points arrêtée pour ${normalizedChannel}`
      );
    }
  }

  // Obtenir la liste des chatters actifs (simulation)
  async getActiveChatters(client, channel) {
    // Dans une implémentation réelle, vous utiliseriez l'API Twitch
    // pour récupérer la liste des chatters

    // Pour cet exemple, nous simulons une liste de chatters
    const simulatedChatters = [
      "viewer1",
      "viewer2",
      "viewer3",
      "viewer4",
      "viewer5",
      "fan1",
      "fan2",
      "subscriber1",
      "subscriber2",
      "mod1",
    ];

    // Simuler des spectateurs aléatoires
    const activeChatters = [];
    const activeCount =
      Math.floor(Math.random() * simulatedChatters.length) + 1;

    for (let i = 0; i < activeCount; i++) {
      activeChatters.push(simulatedChatters[i]);
    }

    return activeChatters;
  }
}

// Créer et exporter une instance unique
const pointsSystem = new PointsSystem();
module.exports = pointsSystem;
```

### 2. Commandes pour le système de points

#### Commande pour consulter ses points (src/commands/points.js)

```javascript
// src/commands/points.js
const pointsSystem = require("../utils/pointsSystem");

module.exports = {
  name: "points",
  description: "Affiche vos points ou ceux d'un autre utilisateur",
  execute: async (client, channel, tags, args) => {
    // Déterminer l'utilisateur cible
    const targetUsername =
      args.length > 0 ? args[0].toLowerCase().replace("@", "") : tags.username;

    // Obtenir les points
    const points = await pointsSystem.getPoints(channel, targetUsername);

    // Déterminer le nom affichable
    const displayName =
      targetUsername === tags.username ? "Vous avez" : `@${targetUsername} a`;

    // Afficher les points
    client.say(
      channel,
      `${displayName} ${points} point${points !== 1 ? "s" : ""}.`
    );
  },
};
```

#### Commande pour le classement (src/commands/classement.js)

```javascript
// src/commands/classement.js
const pointsSystem = require("../utils/pointsSystem");

module.exports = {
  name: "classement",
  description: "Affiche le classement des utilisateurs par points",
  execute: async (client, channel, tags, args) => {
    // Obtenir le classement (top 5 par défaut)
    const limit = args.length > 0 && !isNaN(args[0]) ? parseInt(args[0]) : 5;
    const leaderboard = await pointsSystem.getLeaderboard(
      channel,
      Math.min(limit, 10)
    );

    if (leaderboard.length === 0) {
      return client.say(
        channel,
        `Aucun utilisateur n'a encore de points sur cette chaîne.`
      );
    }

    // Créer le message de classement
    let message = "🏆 Classement des points : ";

    leaderboard.forEach((entry, index) => {
      message += `${index + 1}. ${entry.username} (${entry.points}) `;
    });

    client.say(channel, message);
  },
};
```

#### Commande pour donner des points (src/commands/donner.js)

```javascript
// src/commands/donner.js
const pointsSystem = require("../utils/pointsSystem");

module.exports = {
  name: "donner",
  description: "Donne des points à un autre utilisateur",
  execute: async (client, channel, tags, args) => {
    // Vérifier si les arguments sont valides
    if (args.length < 2 || isNaN(args[1])) {
      return client.say(
        channel,
        `@${tags.username}, utilisation : !donner @utilisateur montant`
      );
    }

    // Extraire l'utilisateur cible et le montant
    const targetUsername = args[0].toLowerCase().replace("@", "");
    const amount = parseInt(args[1]);

    // Vérifier si le montant est positif
    if (amount <= 0) {
      return client.say(
        channel,
        `@${tags.username}, le montant doit être positif.`
      );
    }

    // Empêcher de donner à soi-même
    if (targetUsername === tags.username) {
      return client.say(
        channel,
        `@${tags.username}, vous ne pouvez pas vous donner des points à vous-même.`
      );
    }

    // Vérifier si l'utilisateur a assez de points
    const userPoints = await pointsSystem.getPoints(channel, tags.username);

    if (userPoints < amount) {
      return client.say(
        channel,
        `@${
          tags.username
        }, vous n'avez pas assez de points. Vous avez actuellement ${userPoints} point${
          userPoints !== 1 ? "s" : ""
        }.`
      );
    }

    // Effectuer la transaction
    await pointsSystem.removePoints(channel, tags.username, amount);
    await pointsSystem.addPoints(channel, targetUsername, amount);

    // Confirmer la transaction
    client.say(
      channel,
      `@${tags.username} a donné ${amount} point${
        amount !== 1 ? "s" : ""
      } à @${targetUsername}.`
    );
  },
};
```

### 3. Configuration automatique des points

Ajoutons à notre `index.js` de quoi démarrer automatiquement l'attribution de points :

```javascript
// Ajouter à la fin de src/index.js, après "client.connect()"

// Importer le système de points
const pointsSystem = require("./utils/pointsSystem");

// Quand le bot est connecté, démarrer l'attribution automatique de points
client.on("connected", () => {
  // Pour chaque canal
  const channels = process.env.TWITCH_CHANNELS.split(",");

  channels.forEach((channel) => {
    // Démarrer l'attribution de points toutes les 10 minutes
    pointsSystem.startAutoPointsInterval(client, channel, 10);
  });
});
```

## Intégration avec les API Twitch

Pour créer un bot Twitch complet, vous voudrez probablement interagir avec les API Twitch pour obtenir des informations sur le stream, les abonnés, etc.

### 1. Utilitaire pour l'API Twitch (src/utils/twitchApi.js)

```javascript
// src/utils/twitchApi.js
const fetch = require("node-fetch");

class TwitchAPI {
  constructor() {
    this.clientId = process.env.TWITCH_CLIENT_ID;
    this.clientSecret = process.env.TWITCH_CLIENT_SECRET;
    this.accessToken = null;
    this.tokenExpiry = 0;
  }

  // Obtenir un token d'accès
  async getAccessToken() {
    // Vérifier si le token est encore valide
    if (this.accessToken && Date.now() < this.tokenExpiry) {
      return this.accessToken;
    }

    try {
      // Obtenir un nouveau token
      const response = await fetch(
        `https://id.twitch.tv/oauth2/token?client_id=${this.clientId}&client_secret=${this.clientSecret}&grant_type=client_credentials`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
          },
        }
      );

      const data = await response.json();

      if (!response.ok) {
        throw new Error(`Erreur lors de l'obtention du token: ${data.message}`);
      }

      // Stocker le token et calculer sa date d'expiration
      this.accessToken = data.access_token;
      // Soustraire 10 minutes pour une marge de sécurité
      this.tokenExpiry = Date.now() + (data.expires_in - 600) * 1000;

      return this.accessToken;
    } catch (error) {
      console.error("Erreur lors de l'authentification Twitch:", error);
      throw error;
    }
  }

  // Méthode générique pour effectuer des requêtes API
  async makeRequest(endpoint, params = {}) {
    try {
      const token = await this.getAccessToken();

      // Construire l'URL avec les paramètres
      const url = new URL(`https://api.twitch.tv/helix/${endpoint}`);
      Object.keys(params).forEach((key) =>
        url.searchParams.append(key, params[key])
      );

      // Effectuer la requête
      const response = await fetch(url, {
        headers: {
          "Client-ID": this.clientId,
          Authorization: `Bearer ${token}`,
        },
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(`Erreur API Twitch: ${data.message}`);
      }

      return data;
    } catch (error) {
      console.error(`Erreur lors de la requête à l'API Twitch:`, error);
      throw error;
    }
  }

  // Vérifier si un stream est en ligne
  async getStreamInfo(username) {
    return this.makeRequest("streams", { user_login: username });
  }

  // Obtenir des informations sur un utilisateur
  async getUserInfo(username) {
    return this.makeRequest("users", { login: username });
  }

  // Obtenir des informations sur les abonnés
  async getSubscribers(broadcasterId) {
    return this.makeRequest("subscriptions", { broadcaster_id: broadcasterId });
  }

  // Obtenir des informations sur les followers
  async getFollowers(broadcasterId) {
    return this.makeRequest("channels/followers", {
      broadcaster_id: broadcasterId,
    });
  }

  // Obtenir des statistiques du canal
  async getChannelStats(broadcasterId) {
    return this.makeRequest("channels", { broadcaster_id: broadcasterId });
  }
}

// Créer et exporter une instance unique
const twitchAPI = new TwitchAPI();
module.exports = twitchAPI;
```

### 2. Amélioration de la commande uptime

Maintenant, améliorons notre commande `uptime` pour utiliser l'API Twitch :

```javascript
// src/commands/uptime.js (mise à jour)
const twitchAPI = require("../utils/twitchApi");

module.exports = {
  name: "uptime",
  description: "Affiche depuis combien de temps le stream est en ligne",
  execute: async (client, channel, tags, args) => {
    try {
      // Le nom du canal sans le # au début
      const channelName = channel.slice(1);

      // Obtenir les infos du stream
      const streamData = await twitchAPI.getStreamInfo(channelName);

      if (!streamData.data || streamData.data.length === 0) {
        return client.say(
          channel,
          `${channelName} n'est pas en ligne actuellement.`
        );
      }

      // Calculer l'uptime
      const startTime = new Date(streamData.data[0].started_at);
      const uptime = calculateUptime(startTime);

      client.say(
        channel,
        `${channelName} est en ligne depuis ${uptime}. Catégorie: ${streamData.data[0].game_name}`
      );
    } catch (error) {
      console.error(
        "Erreur lors de la récupération des informations du stream:",
        error
      );
      client.say(
        channel,
        `Impossible de récupérer les informations du stream.`
      );
    }
  },
};

// Fonction pour calculer le temps d'uptime
function calculateUptime(startTime) {
  const currentTime = new Date();
  const diffInMs = currentTime - startTime;

  const hours = Math.floor(diffInMs / (1000 * 60 * 60));
  const minutes = Math.floor((diffInMs % (1000 * 60 * 60)) / (1000 * 60));

  let uptimeStr = "";

  if (hours > 0) {
    uptimeStr += `${hours} heure${hours > 1 ? "s" : ""} `;
  }

  uptimeStr += `${minutes} minute${minutes > 1 ? "s" : ""}`;

  return uptimeStr;
}
```

### 3. Ajout d'une commande pour les followers

```javascript
// src/commands/followers.js
const twitchAPI = require("../utils/twitchApi");

module.exports = {
  name: "followers",
  description: "Affiche le nombre de followers et les derniers abonnés",
  execute: async (client, channel, tags, args) => {
    try {
      // Le nom du canal sans le # au début
      const channelName = channel.slice(1);

      // Obtenir d'abord l'ID du broadcaster
      const userData = await twitchAPI.getUserInfo(channelName);

      if (!userData.data || userData.data.length === 0) {
        return client.say(
          channel,
          `Impossible de trouver les informations de ${channelName}.`
        );
      }

      const broadcasterId = userData.data[0].id;

      // Obtenir les informations des followers
      const followersData = await twitchAPI.getFollowers(broadcasterId);

      // Afficher les résultats
      if (followersData.total === 0) {
        return client.say(
          channel,
          `${channelName} n'a pas encore de followers.`
        );
      }

      // Afficher le nombre total et les derniers followers
      client.say(
        channel,
        `${channelName} a ${followersData.total} follower${
          followersData.total !== 1 ? "s" : ""
        }.`
      );

      // Note: L'API Twitch v5 permettait d'obtenir les derniers followers,
      // mais l'API Helix a une structure différente. Cette partie est donc simplifiée.
    } catch (error) {
      console.error("Erreur lors de la récupération des followers:", error);
      client.say(
        channel,
        `Impossible de récupérer les informations des followers.`
      );
    }
  },
};
```

## Création d'un système de commandes personnalisables

Une fonctionnalité très utile est de permettre aux modérateurs d'ajouter, modifier et supprimer des commandes personnalisées sans avoir à modifier le code.

### 1. Utilitaire pour les commandes personnalisables (src/utils/customCommands.js)

```javascript
// src/utils/customCommands.js
const fs = require("fs").promises;
const path = require("path");

// Chemin vers le fichier de données
const DATA_PATH = path.join(
  __dirname,
  "..",
  "..",
  "data",
  "custom_commands.json"
);

class CustomCommandsManager {
  constructor() {
    this.commands = {};
    this.loaded = false;
  }

  // Charger les commandes depuis le fichier
  async load() {
    try {
      const data = await fs.readFile(DATA_PATH, "utf8");
      this.commands = JSON.parse(data);
    } catch (error) {
      // Si le fichier n'existe pas ou est invalide, créer un objet vide
      this.commands = {};

      // Créer le répertoire de données s'il n'existe pas
      try {
        await fs.mkdir(path.dirname(DATA_PATH), { recursive: true });
      } catch (err) {
        // Ignorer si le dossier existe déjà
      }
    }

    this.loaded = true;
    console.log("Commandes personnalisées chargées");
  }

  // Sauvegarder les commandes dans le fichier
  async save() {
    if (!this.loaded) await this.load();

    try {
      await fs.writeFile(
        DATA_PATH,
        JSON.stringify(this.commands, null, 2),
        "utf8"
      );
    } catch (error) {
      console.error(
        "Erreur lors de la sauvegarde des commandes personnalisées:",
        error
      );
    }
  }

  // Ajouter ou mettre à jour une commande
  async addCommand(channel, commandName, response, addedBy) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Initialiser la structure si nécessaire
    if (!this.commands[normalizedChannel]) {
      this.commands[normalizedChannel] = {};
    }

    // Ajouter/mettre à jour la commande
    this.commands[normalizedChannel][commandName] = {
      response,
      addedBy,
      timestamp: Date.now(),
    };

    // Sauvegarder les changements
    await this.save();

    return true;
  }

  // Supprimer une commande
  async removeCommand(channel, commandName) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si la commande existe
    if (
      !this.commands[normalizedChannel] ||
      !this.commands[normalizedChannel][commandName]
    ) {
      return false;
    }

    // Supprimer la commande
    delete this.commands[normalizedChannel][commandName];

    // Sauvegarder les changements
    await this.save();

    return true;
  }

  // Obtenir une commande
  async getCommand(channel, commandName) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si la commande existe
    if (
      !this.commands[normalizedChannel] ||
      !this.commands[normalizedChannel][commandName]
    ) {
      return null;
    }

    return this.commands[normalizedChannel][commandName];
  }

  // Obtenir toutes les commandes d'un canal
  async getAllCommands(channel) {
    if (!this.loaded) await this.load();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si le canal a des commandes
    if (!this.commands[normalizedChannel]) {
      return {};
    }

    return this.commands[normalizedChannel];
  }

  // Traiter un message pour vérifier s'il contient une commande personnalisée
  async processMessage(client, channel, tags, message, prefix) {
    if (!this.loaded) await this.load();

    // Vérifier si le message commence par le préfixe
    if (!message.startsWith(prefix)) return false;

    // Extraire le nom de la commande
    const commandName = message
      .slice(prefix.length)
      .split(" ")[0]
      .toLowerCase();

    // Normaliser le nom du canal
    const normalizedChannel = channel.startsWith("#")
      ? channel.slice(1)
      : channel;

    // Vérifier si la commande existe
    if (
      !this.commands[normalizedChannel] ||
      !this.commands[normalizedChannel][commandName]
    ) {
      return false;
    }

    // Obtenir la réponse
    const commandData = this.commands[normalizedChannel][commandName];

    // Traiter les variables dans la réponse
    let response = commandData.response;

    // Remplacer les variables
    response = response.replace(/{username}/g, tags.username);
    response = response.replace(/{channel}/g, normalizedChannel);
    response = response.replace(/{random(\d+)?}/g, (match, max) => {
      const maximum = max ? parseInt(max) : 100;
      return Math.floor(Math.random() * maximum) + 1;
    });

    // Vous pouvez ajouter d'autres remplacements de variables ici

    // Envoyer la réponse
    client.say(channel, response);

    return true;
  }
}

// Créer et exporter une instance unique
const customCommandsManager = new CustomCommandsManager();
module.exports = customCommandsManager;
```

### 2. Intégration dans index.js

Modifions notre gestionnaire de messages dans `index.js` pour inclure les commandes personnalisables :

```javascript
// Modifier le gestionnaire de messages dans src/index.js
const customCommandsManager = require("./utils/customCommands");

// Événement : Message reçu (mise à jour)
client.on("message", async (channel, tags, message, self) => {
  // Ignorer les messages du bot lui-même
  if (self) return;

  // Préfixe des commandes
  const prefix = "!";

  // Vérifier si c'est une commande personnalisée
  const isCustomCommand = await customCommandsManager.processMessage(
    client,
    channel,
    tags,
    message,
    prefix
  );

  // Si c'était une commande personnalisée, ne pas continuer
  if (isCustomCommand) return;

  // Vérifier si le message commence par le préfixe
  if (!message.startsWith(prefix)) return;

  // Extraire le nom de la commande et les arguments
  const args = message.slice(prefix.length).trim().split(/\s+/);
  const commandName = args.shift().toLowerCase();

  // Vérifier si la commande existe
  if (!commands.has(commandName)) return;

  // Exécuter la commande
  try {
    await commands.get(commandName).execute(client, channel, tags, args);
  } catch (error) {
    console.error(
      `Erreur lors de l'exécution de la commande ${commandName}:`,
      error
    );
    client.say(
      channel,
      `Une erreur s'est produite lors de l'exécution de cette commande.`
    );
  }
});
```

### 3. Commandes pour gérer les commandes personnalisables

#### Commande pour ajouter/modifier une commande personnalisée (src/commands/addcom.js)

```javascript
// src/commands/addcom.js
const customCommandsManager = require("../utils/customCommands");

module.exports = {
  name: "addcom",
  description: "Ajoute ou modifie une commande personnalisée",
  execute: async (client, channel, tags, args) => {
    // Vérifier si l'utilisateur est modérateur ou streamer
    const isMod = tags.mod || tags.username === channel.slice(1);

    if (!isMod) {
      return client.say(
        channel,
        `@${tags.username}, seuls les modérateurs peuvent gérer les commandes.`
      );
    }

    // Vérifier si les arguments sont valides
    if (args.length < 2) {
      return client.say(
        channel,
        `@${tags.username}, utilisation : !addcom commandName Réponse de la commande`
      );
    }

    // Extraire le nom de la commande et la réponse
    const commandName = args[0].toLowerCase();
    const response = args.slice(1).join(" ");

    // Vérifier que le nom de la commande ne correspond pas à une commande existante du bot
    const reservedCommands = [
      "addcom",
      "delcom",
      "commands",
      "ping",
      "uptime",
      "aide",
      "points",
      "classement",
      "donner",
      "social",
      "sondage",
      "followers",
    ];

    if (reservedCommands.includes(commandName)) {
      return client.say(
        channel,
        `@${tags.username}, impossible d'utiliser le nom "${commandName}" car c'est une commande réservée.`
      );
    }

    // Ajouter la commande
    await customCommandsManager.addCommand(
      channel,
      commandName,
      response,
      tags.username
    );

    client.say(
      channel,
      `@${tags.username}, la commande !${commandName} a été ajoutée avec succès.`
    );
  },
};
```

#### Commande pour supprimer une commande personnalisée (src/commands/delcom.js)

```javascript
// src/commands/delcom.js
const customCommandsManager = require("../utils/customCommands");

module.exports = {
  name: "delcom",
  description: "Supprime une commande personnalisée",
  execute: async (client, channel, tags, args) => {
    // Vérifier si l'utilisateur est modérateur ou streamer
    const isMod = tags.mod || tags.username === channel.slice(1);

    if (!isMod) {
      return client.say(
        channel,
        `@${tags.username}, seuls les modérateurs peuvent gérer les commandes.`
      );
    }

    // Vérifier si les arguments sont valides
    if (args.length < 1) {
      return client.say(
        channel,
        `@${tags.username}, utilisation : !delcom commandName`
      );
    }

    // Extraire le nom de la commande
    const commandName = args[0].toLowerCase();

    // Vérifier si la commande existe
    const command = await customCommandsManager.getCommand(
      channel,
      commandName
    );

    if (!command) {
      return client.say(
        channel,
        `@${tags.username}, la commande !${commandName} n'existe pas.`
      );
    }

    // Supprimer la commande
    await customCommandsManager.removeCommand(channel, commandName);

    client.say(
      channel,
      `@${tags.username}, la commande !${commandName} a été supprimée.`
    );
  },
};
```

#### Commande pour lister les commandes personnalisées (src/commands/commands.js)

```javascript
// src/commands/commands.js
const customCommandsManager = require("../utils/customCommands");

module.exports = {
  name: "commands",
  description: "Affiche la liste des commandes personnalisées",
  execute: async (client, channel, tags, args) => {
    // Obtenir toutes les commandes
    const commands = await customCommandsManager.getAllCommands(channel);

    // Vérifier s'il y a des commandes
    const commandNames = Object.keys(commands);

    if (commandNames.length === 0) {
      return client.say(
        channel,
        `Aucune commande personnalisée n'est définie pour cette chaîne.`
      );
    }

    // Afficher la liste des commandes
    client.say(
      channel,
      `Commandes personnalisées : !${commandNames.join(" !")}`
    );
  },
};
```

## Ajout d'un système de temporisation (cooldown)

Pour éviter le spam de commandes, ajoutons un système de temporisation :

### 1. Utilitaire pour gérer les cooldowns (src/utils/cooldownManager.js)

```javascript
// src/utils/cooldownManager.js
class CooldownManager {
  constructor() {
    // Structure: { commandName: { userId: timestamp } }
    this.cooldowns = new Map();

    // Cooldowns par défaut (en secondes)
    this.defaultCooldowns = {
      global: 3, // Cooldown global entre commandes
      command: 10, // Cooldown par défaut pour une commande spécifique
      user: 5, // Cooldown par défaut pour un utilisateur spécifique
    };
  }

  // Vérifier si une commande est en cooldown pour un utilisateur
  checkCooldown(commandName, userId) {
    const now = Date.now();

    // Vérifier le cooldown global de l'utilisateur
    const userCooldowns = this.cooldowns.get("user_" + userId);
    if (
      userCooldowns &&
      now - userCooldowns < this.defaultCooldowns.user * 1000
    ) {
      return {
        onCooldown: true,
        remainingTime: Math.ceil(
          (userCooldowns + this.defaultCooldowns.user * 1000 - now) / 1000
        ),
        type: "user",
      };
    }

    // Vérifier le cooldown spécifique à la commande
    if (!this.cooldowns.has(commandName)) {
      this.cooldowns.set(commandName, new Map());
    }

    const timestamps = this.cooldowns.get(commandName);
    const cooldownAmount = this.getCommandCooldown(commandName) * 1000;

    if (timestamps.has(userId)) {
      const expirationTime = timestamps.get(userId) + cooldownAmount;

      if (now < expirationTime) {
        const timeLeft = Math.ceil((expirationTime - now) / 1000);
        return {
          onCooldown: true,
          remainingTime: timeLeft,
          type: "command",
        };
      }
    }

    // Actualiser les timestamps
    timestamps.set(userId, now);
    this.cooldowns.set("user_" + userId, now);

    // Nettoyer après le cooldown
    setTimeout(() => timestamps.delete(userId), cooldownAmount);

    return { onCooldown: false };
  }

  // Obtenir le cooldown d'une commande
  getCommandCooldown(commandName) {
    // On pourrait implémenter une logique pour des cooldowns personnalisés par commande
    return this.defaultCooldowns.command;
  }

  // Définir un cooldown personnalisé pour une commande
  setCommandCooldown(commandName, seconds) {
    this.defaultCooldowns[commandName] = seconds;
  }

  // Réinitialiser le cooldown d'un utilisateur pour une commande
  resetCooldown(commandName, userId) {
    if (this.cooldowns.has(commandName)) {
      const timestamps = this.cooldowns.get(commandName);
      timestamps.delete(userId);
    }
  }
}

// Créer et exporter une instance unique
const cooldownManager = new CooldownManager();
module.exports = cooldownManager;
```

### 2. Intégration dans index.js

Modifions à nouveau notre gestionnaire de messages pour inclure le système de cooldown :

```javascript
// Ajouter en haut du fichier src/index.js
const cooldownManager = require("./utils/cooldownManager");

// Modifier le gestionnaire de messages
client.on("message", async (channel, tags, message, self) => {
  // Ignorer les messages du bot lui-même
  if (self) return;

  // Préfixe des commandes
  const prefix = "!";

  // Vérifier si c'est une commande personnalisée
  const isCustomCommand = await customCommandsManager.processMessage(
    client,
    channel,
    tags,
    message,
    prefix
  );

  // Si c'était une commande personnalisée, ne pas continuer
  if (isCustomCommand) return;

  // Vérifier si le message commence par le préfixe
  if (!message.startsWith(prefix)) return;

  // Extraire le nom de la commande et les arguments
  const args = message.slice(prefix.length).trim().split(/\s+/);
  const commandName = args.shift().toLowerCase();

  // Vérifier si la commande existe
  if (!commands.has(commandName)) return;

  // Vérifier si l'utilisateur est en cooldown
  const cooldownCheck = cooldownManager.checkCooldown(
    commandName,
    tags.username
  );

  if (cooldownCheck.onCooldown) {
    // Ignorer silencieusement le cooldown si c'est un mod ou le broadcaster
    const isMod = tags.mod || tags.username === channel.slice(1);

    if (!isMod) {
      // Optionnel : informer l'utilisateur du cooldown (peut être commenté pour réduire le spam)
      // client.say(channel, `@${tags.username}, veuillez attendre ${cooldownCheck.remainingTime} secondes avant de réutiliser cette commande.`);
      return;
    }
  }

  // Exécuter la commande
  try {
    await commands.get(commandName).execute(client, channel, tags, args);
  } catch (error) {
    console.error(
      `Erreur lors de l'exécution de la commande ${commandName}:`,
      error
    );
    client.say(
      channel,
      `Une erreur s'est produite lors de l'exécution de cette commande.`
    );
  }
});
```

## Déploiement et maintenance

### Déploiement sur un serveur

Comme pour le bot Discord, vous pouvez déployer votre bot Twitch sur divers services :

1. **Heroku** :

```
# Procfile
worker: node src/index.js
```

2. **Railway, Render, etc.** :
   Ces plateformes fonctionnent de manière similaire à ce que nous avons vu pour le bot Discord.

3. **VPS** :

```bash
npm install -g pm2
pm2 start src/index.js --name "mon-bot-twitch"
pm2 save
pm2 startup
```

### Surveillance et journalisation

Il est important de surveiller votre bot pour vous assurer qu'il fonctionne correctement. Voici quelques bonnes pratiques :

```javascript
// Ajoutez en haut de src/index.js
const fs = require("fs");
const path = require("path");

// Configurer la journalisation
const logsDir = path.join(__dirname, "..", "logs");
if (!fs.existsSync(logsDir)) {
  fs.mkdirSync(logsDir, { recursive: true });
}

// Journalisation des erreurs
process.on("uncaughtException", (error) => {
  const timestamp = new Date().toISOString();
  const logMessage = `[${timestamp}] Uncaught Exception: ${error.message}\n${error.stack}\n\n`;

  fs.appendFileSync(path.join(logsDir, "error.log"), logMessage);
  console.error("Uncaught Exception:", error);

  // Redémarrer le bot
  process.exit(1);
});

process.on("unhandledRejection", (reason, promise) => {
  const timestamp = new Date().toISOString();
  const logMessage = `[${timestamp}] Unhandled Rejection: ${reason}\n${JSON.stringify(
    promise
  )}\n\n`;

  fs.appendFileSync(path.join(logsDir, "error.log"), logMessage);
  console.error("Unhandled Rejection:", reason);
});
```

## Conseils pour améliorer votre bot Twitch

1. **Automatisation des annonces** :

   - Messages périodiques pour rappeler le follow, la chaîne YouTube, etc.

2. **Modération automatique** :

   - Filtre des liens, majuscules excessives, spam, etc.
   - Timeout ou ban automatique selon les règles définies

3. **Intégrations externes** :

   - Connexion à des services comme Streamlabs, StreamElements
   - Intégration avec des bases de données pour stocker des données persistantes (MongoDB, SQLite)

4. **Internationalisation** :

   - Support de plusieurs langues pour les commandes et réponses

5. **Interface web d'administration** :
   - Création d'un panneau de contrôle pour gérer le bot sans avoir à modifier le code

## Conclusion

Dans ce chapitre, nous avons appris à créer un bot Twitch complet avec Node.js et tmi.js. Nous avons exploré :

- La configuration d'une application Twitch et l'obtention des identifiants
- La création d'un bot de base répondant à des commandes
- L'implémentation d'un système de points et de fidélité
- L'interaction avec l'API Twitch pour obtenir des informations sur le stream
- La création d'un système de commandes personnalisables
- L'ajout d'un système de temporisation pour éviter le spam
- Les meilleures pratiques pour le déploiement et la maintenance

Votre bot Twitch est maintenant prêt à améliorer l'expérience de vos streams et à engager votre communauté.

## Ressources supplémentaires

- [Documentation tmi.js](https://github.com/tmijs/docs/tree/gh-pages/_posts/v1.4.2)
- [API Twitch](https://dev.twitch.tv/docs/api/)
- [Guide des bots Twitch](https://dev.twitch.tv/docs/irc)
- [Communauté Twitch Dev](https://discuss.dev.twitch.tv/)
