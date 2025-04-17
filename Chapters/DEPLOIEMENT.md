# Chapitre 8 - Déploiement et maintenance des bots

Dans ce chapitre, nous allons explorer les aspects essentiels du déploiement et de la maintenance de vos bots Discord et Twitch. L'objectif est de vous montrer comment mettre votre bot en production de manière fiable, le maintenir opérationnel 24h/24, et le faire évoluer au fil du temps.

## Préparation au déploiement

Avant de déployer votre bot, plusieurs préparatifs sont nécessaires pour s'assurer que tout fonctionnera correctement en production.

### Configuration des variables d'environnement

Les variables d'environnement permettent de séparer la configuration sensible du code source. Pour un bot, cela concerne principalement les tokens et clés API.

Créez un fichier `.env.example` qui servira de modèle sans contenir de vraies valeurs :

```
# Discord Bot Configuration
DISCORD_TOKEN=your_discord_token_here
DISCORD_CLIENT_ID=your_client_id_here
DISCORD_GUILD_ID=your_guild_id_here  # Optional for development

# Twitch Bot Configuration
TWITCH_USERNAME=your_twitch_username_here
TWITCH_OAUTH_TOKEN=your_twitch_oauth_token_here
TWITCH_CLIENT_ID=your_twitch_client_id_here
TWITCH_CLIENT_SECRET=your_twitch_client_secret_here
TWITCH_CHANNELS=channel1,channel2,channel3

# Database Configuration
DB_TYPE=sqlite  # or mongodb, mysql, etc.
DB_PATH=./data/bot.db
# MongoDB settings
MONGODB_URI=mongodb://localhost:27017/botdb

# External APIs
OPENWEATHERMAP_API_KEY=your_openweathermap_api_key_here

# Logging
LOG_LEVEL=info  # debug, info, warn, error

# Production Settings
NODE_ENV=production
PORT=3000  # For potential web dashboards
```

La vraie configuration sera dans un fichier `.env` qui ne sera pas partagé dans votre dépôt Git.

### Scripts NPM pour le déploiement

Ajoutez des scripts utiles dans votre `package.json` :

```json
{
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "deploy-commands": "node src/utils/deployCommands.js",
    "lint": "eslint src/",
    "test": "jest",
    "build": "npm run lint && npm run test && npm run deploy-commands",
    "prestart": "npm run build"
  }
}
```

### Gestion des dépendances

Assurez-vous de bien spécifier les versions des dépendances dans votre `package.json` :

```json
"dependencies": {
  "discord.js": "^14.11.0",
  "tmi.js": "^1.8.5",
  "node-fetch": "^2.6.11",
  "dotenv": "^16.0.3",
  "better-sqlite3": "^8.4.0",
  "mongoose": "^7.2.2",
  "@discordjs/voice": "^0.16.0",
  "winston": "^3.8.2"
}
```

Créez un fichier `package-lock.json` pour bloquer les versions des dépendances :

```bash
npm ci
```

### Optimisation du code pour la production

Quelques bonnes pratiques à appliquer avant le déploiement :

1. **Système de logging** - Implémentons un système de logs robuste avec Winston :

```javascript
// src/utils/logger.js
const winston = require("winston");
const path = require("path");
const fs = require("fs");

// Créer le dossier des logs s'il n'existe pas
const logDir = path.join(__dirname, "..", "..", "logs");
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir, { recursive: true });
}

// Configuration des niveaux de log personnalisés
const levels = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  verbose: 4,
  debug: 5,
};

// Configuration des couleurs pour la console
const colors = {
  error: "red",
  warn: "yellow",
  info: "green",
  http: "magenta",
  verbose: "cyan",
  debug: "white",
};

// Ajouter les couleurs à Winston
winston.addColors(colors);

// Format de log personnalisé
const format = winston.format.combine(
  winston.format.timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
  winston.format.errors({ stack: true }),
  winston.format.splat(),
  winston.format.printf(
    (info) =>
      `[${info.timestamp}] [${info.level.toUpperCase()}] ${info.message}${
        info.stack ? `\n${info.stack}` : ""
      }`
  )
);

// Détermine le niveau de log en fonction de l'environnement
const level =
  process.env.LOG_LEVEL ||
  (process.env.NODE_ENV === "production" ? "info" : "debug");

// Créer le logger
const logger = winston.createLogger({
  level,
  levels,
  format,
  transports: [
    // Logs d'erreur dans un fichier séparé
    new winston.transports.File({
      filename: path.join(logDir, "error.log"),
      level: "error",
    }),
    // Logs combinés
    new winston.transports.File({
      filename: path.join(logDir, "combined.log"),
    }),
    // Logs de débogage pour le développement
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
  ],
  // Gérer les exceptions non capturées
  exceptionHandlers: [
    new winston.transports.File({
      filename: path.join(logDir, "exceptions.log"),
    }),
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
  ],
  // Gérer les rejets de promesse non traités
  rejectionHandlers: [
    new winston.transports.File({
      filename: path.join(logDir, "rejections.log"),
    }),
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
  ],
  exitOnError: false,
});

module.exports = logger;
```

2. **Gestion robuste des erreurs** - Mettons à jour notre code pour utiliser le nouveau système de logging :

```javascript
// Ajouter en haut de src/index.js
const logger = require("./utils/logger");

// Remplacer les console.log par logger.info, console.error par logger.error, etc.

// Gestion globale des erreurs non capturées
process.on("uncaughtException", (error) => {
  logger.error("Uncaught Exception:", error);
  // Redémarrage contrôlé
  setTimeout(() => {
    logger.info("Shutting down due to uncaught exception");
    process.exit(1);
  }, 3000);
});

process.on("unhandledRejection", (reason, promise) => {
  logger.error("Unhandled Rejection at:", promise, "reason:", reason);
  // Pas besoin de redémarrer ici, Winston s'en occupe déjà
});
```

## Options de déploiement

Plusieurs options s'offrent à vous pour héberger votre bot, chacune avec ses propres avantages et inconvénients.

### Hébergement sur VPS (Virtual Private Server)

Un VPS vous donne un contrôle total sur l'environnement mais nécessite plus de configuration manuelle.

#### Configuration d'un VPS avec Ubuntu

1. **Mise à jour du système** :

```bash
sudo apt update
sudo apt upgrade -y
```

2. **Installation de Node.js** :

```bash
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt-get install -y nodejs
```

3. **Installation de PM2** pour gérer les processus :

```bash
sudo npm install -g pm2
```

4. **Configuration du pare-feu** :

```bash
sudo apt install ufw
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

5. **Installation de Git** :

```bash
sudo apt install git
```

6. **Clonage du dépôt du bot** :

```bash
git clone https://github.com/votre-compte/votre-bot.git
cd votre-bot
```

7. **Configuration des variables d'environnement** :

```bash
cp .env.example .env
nano .env  # Éditer avec vos vraies valeurs
```

8. **Installation des dépendances** :

```bash
npm ci
```

9. **Lancement du bot avec PM2** :

```bash
pm2 start src/index.js --name "discord-bot"
pm2 save
pm2 startup
```

10. **Monitoring avec PM2** :

```bash
pm2 status
pm2 logs
pm2 monit
```

### Déploiement sur les plateformes PaaS

Les plateformes PaaS (Platform as a Service) comme Heroku, Railway, ou Render simplifient grandement le déploiement.

#### Heroku

1. **Création d'un fichier `Procfile`** :

```
worker: node src/index.js
```

2. **Configuration pour Heroku** :

```bash
heroku create votre-bot-discord
heroku config:set DISCORD_TOKEN=your_token_here
heroku config:set [...] # Autres variables d'environnement
git push heroku main
heroku ps:scale worker=1
```

#### Railway

Railway est devenu une option populaire pour héberger des bots Discord :

1. Créez un compte sur [Railway](https://railway.app/).
2. Connectez votre compte GitHub.
3. Sélectionnez votre dépôt.
4. Configurez les variables d'environnement dans l'interface Railway.
5. Ajoutez la commande de démarrage : `npm start`.

#### GitHub Actions + DigitalOcean

Pour un déploiement automatisé plus avancé :

1. **Création d'un fichier de workflow GitHub Actions** dans `.github/workflows/deploy.yml` :

```yaml
name: Deploy Bot

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "16"

      - name: Install Dependencies
        run: npm ci

      - name: Run Tests
        run: npm test

      - name: Deploy to DigitalOcean
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /path/to/your/bot
            git pull
            npm ci
            npm run build
            pm2 restart discord-bot
```

2. **Configuration des secrets GitHub** dans les paramètres de votre dépôt.

### Docker

Docker permet de conteneuriser votre application pour un déploiement uniforme sur différentes plateformes.

1. **Création d'un `Dockerfile`** :

```Dockerfile
FROM node:16-alpine

# Créer le répertoire de l'application
WORKDIR /usr/src/app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installation des dépendances
RUN npm ci --only=production

# Copier le code source
COPY . .

# Créer un utilisateur non-root
RUN adduser -D botuser
USER botuser

# Démarrer l'application
CMD [ "node", "src/index.js" ]
```

2. **Création d'un fichier `.dockerignore`** :

```
node_modules
npm-debug.log
.env
.git
.gitignore
README.md
logs/
data/
```

3. **Construction et exécution du conteneur** :

```bash
docker build -t discord-bot .
docker run -d --name discord-bot --env-file .env discord-bot
```

4. **Utilisation de Docker Compose** pour une configuration multi-conteneurs (par exemple avec une base de données) :

```yaml
# docker-compose.yml
version: "3"
services:
  bot:
    build: .
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./logs:/usr/src/app/logs
      - ./data:/usr/src/app/data
    depends_on:
      - db

  db:
    image: mongo:5
    restart: unless-stopped
    volumes:
      - mongodb_data:/data/db
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=example

volumes:
  mongodb_data:
```

5. **Lancement avec Docker Compose** :

```bash
docker-compose up -d
```

## Surveillance et maintenance

### Surveillance avec PM2

PM2 offre plusieurs outils de surveillance :

```bash
# Interface de monitoring en temps réel
pm2 monit

# Tableaux de bord Web (avec module pm2-web)
pm2 install pm2-web

# Configuration de notifications par e-mail
pm2 install pm2-notify
```

### Surveillance automatisée avec Healthchecks.io

Healthchecks.io permet de s'assurer que votre bot fonctionne correctement :

1. **Création d'un compte et d'un "check"** sur [Healthchecks.io](https://healthchecks.io/).

2. **Implémentation dans le bot** :

```javascript
// src/utils/healthcheck.js
const fetch = require("node-fetch");
const logger = require("./logger");

class HealthCheck {
  constructor(url) {
    this.url = url;
    this.interval = null;
  }

  // Démarrer les pings réguliers
  start(intervalMinutes = 5) {
    if (!this.url) {
      logger.warn("Healthcheck URL not provided, skipping healthchecks");
      return;
    }

    // Arrêter l'intervalle existant s'il y en a un
    this.stop();

    // Envoyer un ping immédiatement
    this.ping();

    // Définir un intervalle pour les pings réguliers
    this.interval = setInterval(() => {
      this.ping();
    }, intervalMinutes * 60 * 1000);

    logger.info(
      `Healthcheck started, pinging every ${intervalMinutes} minutes`
    );
  }

  // Arrêter les pings
  stop() {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
      logger.info("Healthcheck stopped");
    }
  }

  // Envoyer un ping
  async ping() {
    if (!this.url) return;

    try {
      const response = await fetch(this.url);
      logger.debug(`Healthcheck ping sent, response: ${response.status}`);
    } catch (error) {
      logger.error("Error sending healthcheck ping:", error);
    }
  }
}

// Créer et exporter une instance avec l'URL de Healthchecks.io
const healthCheck = new HealthCheck(process.env.HEALTHCHECK_URL);
module.exports = healthCheck;
```

3. **Utilisation dans `index.js`** :

```javascript
const healthCheck = require("./utils/healthcheck");

// Après avoir initialisé et connecté le bot
client.once("ready", () => {
  logger.info(`Logged in as ${client.user.tag}!`);
  // Démarrer les healthchecks
  healthCheck.start(5); // Ping toutes les 5 minutes
});

// S'assurer d'arrêter correctement
process.on("SIGINT", () => {
  healthCheck.stop();
  // Autres opérations de nettoyage...
  process.exit(0);
});
```

### Tableau de bord d'administration Web

Un tableau de bord Web peut vous aider à gérer votre bot à distance. Voici les bases d'un serveur Express simple :

```javascript
// src/dashboard/server.js
const express = require("express");
const session = require("express-session");
const passport = require("passport");
const { Strategy } = require("passport-discord");
const path = require("path");
const logger = require("../utils/logger");

// Configuration
const PORT = process.env.PORT || 3000;
const CLIENT_ID = process.env.DISCORD_CLIENT_ID;
const CLIENT_SECRET = process.env.DISCORD_CLIENT_SECRET;
const CALLBACK_URL =
  process.env.DASHBOARD_CALLBACK_URL ||
  `http://localhost:${PORT}/auth/discord/callback`;
const SESSION_SECRET = process.env.SESSION_SECRET || "keyboard cat";

// Créer l'application Express
const app = express();

// Configuration de la session
app.use(
  session({
    secret: SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: { secure: process.env.NODE_ENV === "production" },
  })
);

// Configuration de Passport
passport.serializeUser((user, done) => done(null, user));
passport.deserializeUser((obj, done) => done(null, obj));

passport.use(
  new Strategy(
    {
      clientID: CLIENT_ID,
      clientSecret: CLIENT_SECRET,
      callbackURL: CALLBACK_URL,
      scope: ["identify", "guilds"],
    },
    (accessToken, refreshToken, profile, done) => {
      // Vous pouvez stocker l'utilisateur dans une base de données ici
      return done(null, profile);
    }
  )
);

app.use(passport.initialize());
app.use(passport.session());

// Middleware pour vérifier l'authentification
function isAuthenticated(req, res, next) {
  if (req.isAuthenticated()) {
    return next();
  }
  res.redirect("/login");
}

// Routes d'authentification
app.get("/login", (req, res) => {
  res.send('Login page <a href="/auth/discord">Login with Discord</a>');
});

app.get("/auth/discord", passport.authenticate("discord"));

app.get(
  "/auth/discord/callback",
  passport.authenticate("discord", { failureRedirect: "/login" }),
  (req, res) => {
    res.redirect("/dashboard");
  }
);

app.get("/logout", (req, res) => {
  req.logout();
  res.redirect("/");
});

// Routes du tableau de bord
app.get("/", (req, res) => {
  res.send('Welcome to the bot dashboard! <a href="/login">Login</a>');
});

app.get("/dashboard", isAuthenticated, (req, res) => {
  res.send(`Welcome, ${req.user.username}! <a href="/logout">Logout</a>`);
});

// API pour les informations sur le bot
app.get("/api/stats", isAuthenticated, (req, res) => {
  // Récupérer les statistiques du bot
  const stats = {
    uptime: process.uptime(),
    memoryUsage: process.memoryUsage(),
    // Ajouter d'autres statistiques du bot ici
  };

  res.json(stats);
});

// Servir les fichiers statiques (pour une interface utilisateur plus riche)
app.use(express.static(path.join(__dirname, "public")));

// Démarrer le serveur
app.listen(PORT, () => {
  logger.info(`Dashboard running at http://localhost:${PORT}`);
});

module.exports = app;
```

### Sauvegardes régulières

Mettez en place un système de sauvegarde pour éviter la perte de données :

```javascript
// src/utils/backup.js
const fs = require("fs").promises;
const path = require("path");
const { exec } = require("child_process");
const util = require("util");
const execPromise = util.promisify(exec);
const logger = require("./logger");

class BackupSystem {
  constructor(options = {}) {
    this.dataDir = options.dataDir || path.join(__dirname, "..", "..", "data");
    this.backupDir =
      options.backupDir || path.join(__dirname, "..", "..", "backups");
    this.databaseFile = options.databaseFile || "bot.db";
    this.maxBackups = options.maxBackups || 7; // Nombre de sauvegardes à conserver
  }

  // Créer une sauvegarde
  async createBackup() {
    try {
      // S'assurer que le répertoire de sauvegarde existe
      await fs.mkdir(this.backupDir, { recursive: true });

      // Générer un nom de fichier basé sur la date
      const date = new Date();
      const timestamp = date
        .toISOString()
        .replace(/:/g, "-")
        .replace(/\..+/, "");
      const backupFileName = `backup_${timestamp}.tar.gz`;
      const backupPath = path.join(this.backupDir, backupFileName);

      // Créer l'archive
      await execPromise(
        `tar -czf ${backupPath} -C ${path.dirname(
          this.dataDir
        )} ${path.basename(this.dataDir)}`
      );

      logger.info(`Backup created: ${backupFileName}`);

      // Supprimer les anciennes sauvegardes si nécessaire
      await this.cleanOldBackups();

      return backupPath;
    } catch (error) {
      logger.error("Error creating backup:", error);
      throw error;
    }
  }

  // Nettoyer les anciennes sauvegardes
  async cleanOldBackups() {
    try {
      // Lister toutes les sauvegardes
      const files = await fs.readdir(this.backupDir);

      // Filtrer uniquement les fichiers de sauvegarde
      const backups = files
        .filter(
          (file) => file.startsWith("backup_") && file.endsWith(".tar.gz")
        )
        .map((file) => ({
          name: file,
          path: path.join(this.backupDir, file),
          time: fs
            .stat(path.join(this.backupDir, file))
            .then((stat) => stat.mtime.getTime()),
        }));

      // Attendre que toutes les statistiques soient chargées
      const backupsWithTime = await Promise.all(
        backups.map(async (backup) => ({
          ...backup,
          time: await backup.time,
        }))
      );

      // Trier par date (les plus récents d'abord)
      backupsWithTime.sort((a, b) => b.time - a.time);

      // Supprimer les sauvegardes excédentaires
      const backupsToDelete = backupsWithTime.slice(this.maxBackups);

      for (const backup of backupsToDelete) {
        await fs.unlink(backup.path);
        logger.info(`Deleted old backup: ${backup.name}`);
      }
    } catch (error) {
      logger.error("Error cleaning old backups:", error);
      throw error;
    }
  }

  // Restaurer à partir d'une sauvegarde
  async restoreFromBackup(backupFile) {
    try {
      const backupPath = path.join(this.backupDir, backupFile);

      // Vérifier si le fichier de sauvegarde existe
      await fs.access(backupPath);

      // Créer un répertoire temporaire
      const tempDir = path.join(this.backupDir, "temp_restore");
      await fs.mkdir(tempDir, { recursive: true });

      // Extraire la sauvegarde
      await execPromise(`tar -xzf ${backupPath} -C ${tempDir}`);

      // Copier les données restaurées
      await execPromise(
        `cp -R ${path.join(tempDir, path.basename(this.dataDir))}/* ${
          this.dataDir
        }/`
      );

      // Nettoyer le répertoire temporaire
      await execPromise(`rm -rf ${tempDir}`);

      logger.info(`Restored from backup: ${backupFile}`);

      return true;
    } catch (error) {
      logger.error("Error restoring from backup:", error);
      throw error;
    }
  }

  // Planifier des sauvegardes régulières
  scheduleBackups(intervalHours = 24) {
    // Convertir les heures en millisecondes
    const intervalMs = intervalHours * 60 * 60 * 1000;

    // Exécuter une sauvegarde immédiatement
    this.createBackup().catch((err) =>
      logger.error("Error in initial backup:", err)
    );

    // Planifier les sauvegardes régulières
    setInterval(() => {
      this.createBackup().catch((err) =>
        logger.error("Error in scheduled backup:", err)
      );
    }, intervalMs);

    logger.info(`Scheduled backups every ${intervalHours} hours`);
  }
}

// Créer et exporter une instance
const backupSystem = new BackupSystem();
module.exports = backupSystem;
```

Utilisation dans `index.js` :

```javascript
const backupSystem = require("./utils/backup");

// Après avoir initialisé le bot
client.once("ready", () => {
  // ...

  // Planifier des sauvegardes quotidiennes
  backupSystem.scheduleBackups(24);
});
```

## Scalabilité et évolution

### Architecture pour les grands bots

Pour les bots qui doivent fonctionner à grande échelle (présents sur des milliers de serveurs), une architecture plus avancée peut être nécessaire :

```
                  ┌───────────────┐
                  │  Load Balancer│
                  └───────┬───────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
┌─────────▼─────────┐ ┌───▼───────────┐ ┌─▼─────────────┐
│  Bot Instance 1   │ │  Bot Instance 2│ │  Bot Instance 3│
└─────────┬─────────┘ └───┬───────────┘ └─┬─────────────┘
          │               │               │
          └───────────────┼───────────────┘
                          │
                ┌─────────▼────────┐
                │ Database Cluster │
                └──────────────────┘
```

#### Sharding avec Discord.js

Pour les bots Discord présents sur plus de 2 500 serveurs, le sharding devient obligatoire :

```javascript
// src/sharding.js
const { ShardingManager } = require("discord.js");
const logger = require("./utils/logger");

// Créer un gestionnaire de shards
const manager = new ShardingManager("./src/index.js", {
  token: process.env.DISCORD_TOKEN,
  totalShards: "auto", // Discord.js calculera automatiquement le nombre optimal
  respawn: true,
});

// Gérer les événements du gestionnaire de shards
manager.on("shardCreate", (shard) => {
  logger.info(`Launched shard ${shard.id}`);

  shard.on("ready", () => {
    logger.info(`Shard ${shard.id} ready`);
  });

  shard.on("disconnect", () => {
    logger.warn(`Shard ${shard.id} disconnected`);
  });

  shard.on("reconnecting", () => {
    logger.info(`Shard ${shard.id} reconnecting`);
  });

  shard.on("death", (process) => {
    logger.error(`Shard ${shard.id} died with exit code ${process.exitCode}`);
  });

  shard.on("error", (error) => {
    logger.error(`Shard ${shard.id} error:`, error);
  });
});

// Démarrer le gestionnaire de shards
manager
  .spawn()
  .then(() => {
    logger.info("All shards spawned successfully");
  })
  .catch((error) => {
    logger.error("Failed to spawn shards:", error);
  });
```

#### Communication entre les shards

```javascript
// Dans un fichier de commande (par exemple pour les statistiques globales)
module.exports = {
  data: new SlashCommandBuilder()
    .setName("stats")
    .setDescription("Affiche les statistiques globales du bot"),

  async execute(interaction) {
    await interaction.deferReply();

    try {
      // Collecter les statistiques de tous les shards
      const promises = [
        interaction.client.shard.fetchClientValues("guilds.cache.size"),
        interaction.client.shard.fetchClientValues("channels.cache.size"),
        interaction.client.shard.fetchClientValues("users.cache.size"),
      ];

      const [guildCounts, channelCounts, userCounts] = await Promise.all(
        promises
      );

      // Calculer les totaux
      const totalGuilds = guildCounts.reduce(
        (total, count) => total + count,
        0
      );
      const totalChannels = channelCounts.reduce(
        (total, count) => total + count,
        0
      );
      const totalUsers = userCounts.reduce((total, count) => total + count, 0);

      // Créer et envoyer l'embed
      const embed = new EmbedBuilder()
        .setColor("#0099ff")
        .setTitle("Statistiques globales du bot")
        .addFields(
          { name: "Serveurs", value: totalGuilds.toString(), inline: true },
          { name: "Canaux", value: totalChannels.toString(), inline: true },
          { name: "Utilisateurs", value: totalUsers.toString(), inline: true },
          {
            name: "Shards",
            value: interaction.client.shard.count.toString(),
            inline: true,
          },
          {
            name: "Shard actuel",
            value: interaction.guild.shardId.toString(),
            inline: true,
          },
          {
            name: "Uptime",
            value: formatUptime(interaction.client.uptime),
            inline: true,
          }
        )
        .setFooter({
          text: `Shard ${interaction.guild.shardId}/${interaction.client.shard.count}`,
        });

      await interaction.editReply({ embeds: [embed] });
    } catch (error) {
      logger.error("Error fetching shard statistics:", error);
      await interaction.editReply(
        "Une erreur s'est produite lors de la récupération des statistiques."
      );
    }
  },
};

// Fonction pour formater le temps d'activité
function formatUptime(uptime) {
  const seconds = Math.floor(uptime / 1000) % 60;
  const minutes = Math.floor(uptime / (1000 * 60)) % 60;
  const hours = Math.floor(uptime / (1000 * 60 * 60)) % 24;
  const days = Math.floor(uptime / (1000 * 60 * 60 * 24));

  return `${days}j ${hours}h ${minutes}m ${seconds}s`;
}
```

### Clustering pour Twitch

Pour les bots Twitch qui doivent gérer un grand nombre de canaux :

```javascript
// src/cluster.js
const cluster = require("cluster");
const os = require("os");
const logger = require("./utils/logger");

// Nombre de CPU disponibles
const numCPUs = os.cpus().length;

// Configuration du clustering
if (cluster.isMaster) {
  logger.info(`Master process ${process.pid} is running`);

  // Lister les canaux Twitch
  const allChannels = process.env.TWITCH_CHANNELS.split(",");

  // Calculer combien de canaux par worker
  const numWorkers = Math.min(numCPUs, allChannels.length);
  const channelsPerWorker = Math.ceil(allChannels.length / numWorkers);

  // Créer les workers
  for (let i = 0; i < numWorkers; i++) {
    // Calculer les canaux pour ce worker
    const startIndex = i * channelsPerWorker;
    const endIndex = Math.min(
      startIndex + channelsPerWorker,
      allChannels.length
    );
    const workerChannels = allChannels.slice(startIndex, endIndex);

    // Créer un worker avec ses canaux
    const worker = cluster.fork({
      WORKER_ID: i,
      WORKER_CHANNELS: workerChannels.join(","),
    });

    logger.info(
      `Worker ${i} started (PID: ${worker.process.pid}) handling ${workerChannels.length} channels`
    );
  }

  // Gérer la fin des workers
  cluster.on("exit", (worker, code, signal) => {
    logger.warn(
      `Worker ${worker.process.pid} died with code ${code} and signal ${signal}`
    );
    logger.info("Starting a new worker...");
    cluster.fork();
  });
} else {
  // Code du worker
  logger.info(
    `Worker ${process.env.WORKER_ID} started, handling channels: ${process.env.WORKER_CHANNELS}`
  );

  // Modifier le code du bot pour utiliser uniquement les canaux assignés
  process.env.TWITCH_CHANNELS = process.env.WORKER_CHANNELS;

  // Charger le bot normal
  require("./index");
}
```

## Mise à jour et évolution du bot

### Déploiement continu

Configuration d'un pipeline CI/CD complet avec GitHub Actions :

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "16"

      - name: Install Dependencies
        run: npm ci

      - name: Lint Code
        run: npm run lint

      - name: Run Tests
        run: npm test

  deploy:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "16"

      - name: Install Dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to Production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /path/to/your/bot
            git pull
            npm ci --production
            pm2 restart discord-bot
```

### Stratégies de mise à jour

1. **Mise à jour progressive** - Script d'exemple :

```javascript
// src/utils/updater.js
const { exec } = require("child_process");
const util = require("util");
const execPromise = util.promisify(exec);
const logger = require("./logger");
const backupSystem = require("./backup");

class Updater {
  constructor() {
    this.updateInProgress = false;
  }

  // Vérifier les mises à jour
  async checkForUpdates() {
    if (this.updateInProgress) {
      logger.warn("Update already in progress, skipping check");
      return false;
    }

    try {
      // Fetch les dernières modifications
      const { stdout } = await execPromise("git fetch && git status");

      // Vérifier si nous sommes à jour
      if (stdout.includes("Your branch is up to date")) {
        logger.info("Bot is already up to date");
        return false;
      }

      logger.info("Updates available");
      return true;
    } catch (error) {
      logger.error("Error checking for updates:", error);
      return false;
    }
  }

  // Appliquer les mises à jour
  async update() {
    if (this.updateInProgress) {
      logger.warn("Update already in progress");
      return false;
    }

    this.updateInProgress = true;

    try {
      // Créer une sauvegarde avant la mise à jour
      logger.info("Creating backup before update");
      await backupSystem.createBackup();

      // Pull les dernières modifications
      logger.info("Pulling latest changes");
      await execPromise("git pull");

      // Installer les dépendances
      logger.info("Installing dependencies");
      await execPromise("npm ci --production");

      logger.info("Update completed successfully");

      // Indiquer qu'un redémarrage est nécessaire
      return true;
    } catch (error) {
      logger.error("Error during update:", error);
      return false;
    } finally {
      this.updateInProgress = false;
    }
  }

  // Redémarrer le bot via PM2
  async restart() {
    try {
      logger.info("Restarting bot");
      await execPromise("pm2 restart discord-bot");
      return true;
    } catch (error) {
      logger.error("Error restarting bot:", error);
      return false;
    }
  }

  // Processus complet de mise à jour
  async updateAndRestart() {
    // Vérifier les mises à jour
    const updatesAvailable = await this.checkForUpdates();

    if (!updatesAvailable) {
      return false;
    }

    // Appliquer les mises à jour
    const updateSuccess = await this.update();

    if (!updateSuccess) {
      return false;
    }

    // Redémarrer le bot
    return await this.restart();
  }

  // Planifier des vérifications automatiques
  scheduleAutoUpdates(intervalHours = 12) {
    // Convertir les heures en millisecondes
    const intervalMs = intervalHours * 60 * 60 * 1000;

    // Planifier les vérifications régulières
    setInterval(async () => {
      logger.info("Running scheduled update check");
      await this.updateAndRestart();
    }, intervalMs);

    logger.info(`Scheduled update checks every ${intervalHours} hours`);
  }
}

// Créer et exporter une instance
const updater = new Updater();
module.exports = updater;
```

2. **Commande de mise à jour pour les administrateurs du bot** :

```javascript
// src/commands/update.js (Discord)
const { SlashCommandBuilder } = require("discord.js");
const updater = require("../utils/updater");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("update")
    .setDescription("Met à jour le bot avec la dernière version")
    .addBooleanOption((option) =>
      option
        .setName("force")
        .setDescription(
          "Forcer la mise à jour même s'il n'y a pas de nouvelles modifications"
        )
        .setRequired(false)
    ),

  async execute(interaction) {
    // Vérifier si l'utilisateur est autorisé à mettre à jour le bot
    if (interaction.user.id !== process.env.OWNER_ID) {
      return interaction.reply({
        content: "Vous n'êtes pas autorisé à utiliser cette commande.",
        ephemeral: true,
      });
    }

    await interaction.deferReply();

    const force = interaction.options.getBoolean("force") || false;

    try {
      // Vérifier les mises à jour
      const updatesAvailable = force || (await updater.checkForUpdates());

      if (!updatesAvailable) {
        return interaction.editReply(
          "Le bot est déjà à jour avec la dernière version."
        );
      }

      // Informer que la mise à jour commence
      await interaction.editReply(
        "🔄 Mise à jour en cours... Le bot peut être indisponible pendant quelques instants."
      );

      // Appliquer les mises à jour
      const updateSuccess = await updater.update();

      if (!updateSuccess) {
        return interaction.editReply(
          "❌ La mise à jour a échoué. Consultez les logs pour plus d'informations."
        );
      }

      // Redémarrer le bot
      await interaction.editReply(
        "✅ Mise à jour réussie. Redémarrage du bot..."
      );

      // Prévoir le redémarrage après un court délai
      setTimeout(async () => {
        await updater.restart();
      }, 2000);
    } catch (error) {
      console.error("Error during update command:", error);
      return interaction.editReply(
        "❌ Une erreur s'est produite pendant la mise à jour. Consultez les logs pour plus d'informations."
      );
    }
  },
};
```

### Tests automatisés

Mise en place de tests automatisés pour assurer la qualité du code :

1. **Installation des dépendances** :

```bash
npm install --save-dev jest discord.js-mock
```

2. **Configuration de Jest** dans `package.json` :

```json
"jest": {
  "testEnvironment": "node",
  "testMatch": ["**/__tests__/**/*.js", "**/?(*.)+(spec|test).js"],
  "collectCoverage": true,
  "coverageDirectory": "coverage"
}
```

3. **Exemple de test pour une commande** :

```javascript
// __tests__/commands/ping.test.js
const { SlashCommandBuilder } = require("discord.js");
const pingCommand = require("../../src/commands/ping");

// Mock des objets Discord.js
const mockInteraction = {
  reply: jest.fn().mockResolvedValue({}),
  editReply: jest.fn().mockResolvedValue({}),
  createdTimestamp: Date.now(),
  client: {
    ws: {
      ping: 15,
    },
  },
};

describe("Ping Command", () => {
  // Reset les mocks entre chaque test
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test("should have correct command structure", () => {
    expect(pingCommand.data).toBeInstanceOf(SlashCommandBuilder);
    expect(pingCommand.data.name).toBe("ping");
    expect(typeof pingCommand.execute).toBe("function");
  });

  test("should reply with pong message", async () => {
    // Exécuter la commande
    await pingCommand.execute(mockInteraction);

    // Vérifier que reply a été appelé
    expect(mockInteraction.reply).toHaveBeenCalledTimes(1);
    expect(mockInteraction.reply.mock.calls[0][0]).toContain(
      "Calcul de la latence"
    );

    // Vérifier que editReply a été appelé avec les latences
    expect(mockInteraction.editReply).toHaveBeenCalledTimes(1);
    expect(mockInteraction.editReply.mock.calls[0][0]).toContain("Pong !");
    expect(mockInteraction.editReply.mock.calls[0][0]).toContain("Latence API");
  });
});
```

## Conclusion

Dans ce chapitre, nous avons exploré les aspects essentiels du déploiement et de la maintenance d'un bot Discord ou Twitch en production :

1. **Préparation au déploiement** - Configuration des variables d'environnement, scripts NPM et gestion des dépendances
2. **Options de déploiement** - Déploiement sur VPS, plateformes PaaS comme Heroku ou Railway, et conteneurisation avec Docker
3. **Surveillance et maintenance** - Mise en place de systèmes de logs, surveillance automatisée et sauvegardes régulières
4. **Scalabilité** - Techniques pour faire évoluer votre bot avec le sharding Discord et le clustering pour Twitch
5. **Mise à jour et évolution** - Stratégies de déploiement continu et tests automatisés

Ces pratiques vous aideront à maintenir un bot fiable, performant et évolutif, capable de servir efficacement votre communauté, qu'elle comporte des dizaines ou des milliers d'utilisateurs.

En suivant ces conseils, vous pourrez non seulement déployer votre bot avec succès, mais aussi le maintenir et le faire évoluer sur le long terme.

## Ressources supplémentaires

- [Documentation PM2](https://pm2.keymetrics.io/docs/usage/quick-start/)
- [Documentation Docker](https://docs.docker.com/get-started/)
- [Guides de sharding Discord.js](https://discordjs.guide/sharding/)
- [AWS Lambda pour les bots serverless](https://aws.amazon.com/lambda/)
- [Tutoriel CI/CD avec GitHub Actions](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-nodejs)
- [Healthchecks.io Documentation](https://healthchecks.io/docs/)
- [Documentation Jest](https://jestjs.io/docs/getting-started)
