# Chapitre 6 - Intégration des bases de données

Dans ce chapitre, nous allons explorer comment intégrer des bases de données à vos bots Discord et Twitch. L'utilisation de bases de données permettra à vos bots de stocker des informations de manière persistante, de gérer efficacement de grandes quantités de données et d'offrir des fonctionnalités plus avancées.

## Pourquoi utiliser une base de données ?

Jusqu'à présent, nous avons utilisé des fichiers JSON pour stocker des données comme les points des utilisateurs ou les commandes personnalisées. Bien que cette approche soit simple, elle présente plusieurs limitations :

- **Performance limitée** : Les fichiers JSON deviennent inefficaces avec de grandes quantités de données
- **Pas de requêtes complexes** : Difficile de filtrer, trier ou agréger les données
- **Concurrence limitée** : Risques de corruption des données lors d'accès simultanés
- **Évolutivité restreinte** : Difficile à faire évoluer pour des bots utilisés sur plusieurs serveurs

Les bases de données résolvent ces problèmes en offrant :

- **Stockage optimisé** : Conçues pour gérer efficacement de grandes quantités de données
- **Requêtes puissantes** : Capacité à rechercher, filtrer et agréger des données rapidement
- **Gestion de la concurrence** : Mécanismes intégrés pour gérer les accès simultanés
- **Scalabilité** : Peuvent évoluer avec votre bot

## Types de bases de données pour les bots

### Bases de données SQL

Les bases de données SQL (relationnelles) utilisent des tables structurées avec des relations entre elles.

**Avantages :**

- Structure de données rigide et prévisible
- Intégrité des données garantie
- Requêtes complexes via SQL
- Transactions ACID (Atomicité, Cohérence, Isolation, Durabilité)

**Options populaires :**

- **SQLite** : Légère, fonctionne sans serveur, idéale pour les petits bots
- **MySQL/MariaDB** : Robuste, bonne performance, largement utilisée
- **PostgreSQL** : Très puissante, fonctionnalités avancées

### Bases de données NoSQL

Les bases de données NoSQL offrent une structure plus flexible, souvent basée sur des documents JSON.

**Avantages :**

- Schéma flexible qui peut évoluer
- Mise à l'échelle horizontale facile
- Bonnes performances pour les lectures/écritures simples
- Simplicité de modélisation pour certains types de données

**Options populaires :**

- **MongoDB** : Base de données orientée documents
- **Firebase Firestore** : Solution cloud avec synchronisation en temps réel
- **Redis** : Stockage clé-valeur en mémoire, idéal pour les caches et les données temporaires

## SQLite : Une solution simple pour commencer

Pour ce chapitre, nous allons commencer par SQLite, car c'est une option simple qui ne nécessite pas de serveur de base de données séparé. Elle est parfaite pour les bots de taille moyenne.

### Installation de SQLite avec Node.js

```bash
npm install sqlite3 better-sqlite3
```

Nous installerons à la fois `sqlite3` (l'API traditionnelle asynchrone) et `better-sqlite3` (une implémentation plus performante avec une API synchrone), car ils ont chacun leurs avantages.

### Création d'un gestionnaire de base de données

Commençons par créer un module utilitaire pour gérer notre base de données :

```javascript
// src/utils/database.js
const path = require("path");
const BetterSqlite3 = require("better-sqlite3");

class Database {
  constructor() {
    this.db = null;
    this.initialized = false;
  }

  // Initialiser la connexion à la base de données
  init() {
    if (this.initialized) return;

    // Chemin vers le fichier de base de données
    const dbPath = path.join(__dirname, "..", "..", "data", "bot.db");

    // Créer la connexion à la base de données
    this.db = new BetterSqlite3(dbPath, {
      verbose: process.env.NODE_ENV === "development" ? console.log : null,
    });

    // Activer les contraintes de clé étrangère
    this.db.pragma("foreign_keys = ON");

    this.initialized = true;
    console.log("Base de données initialisée");

    // Créer les tables si elles n'existent pas
    this.createTables();
  }

  // Créer les tables nécessaires
  createTables() {
    // Table des utilisateurs
    this.db.exec(`
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                platform TEXT NOT NULL,
                user_id TEXT NOT NULL,
                username TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                last_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(platform, user_id)
            )
        `);

    // Table des serveurs/canaux
    this.db.exec(`
            CREATE TABLE IF NOT EXISTS channels (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                platform TEXT NOT NULL,
                channel_id TEXT NOT NULL,
                channel_name TEXT NOT NULL,
                prefix TEXT DEFAULT '!',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                UNIQUE(platform, channel_id)
            )
        `);

    // Table des points
    this.db.exec(`
            CREATE TABLE IF NOT EXISTS points (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                channel_id INTEGER NOT NULL,
                points INTEGER DEFAULT 0,
                last_gained TIMESTAMP,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
                FOREIGN KEY (channel_id) REFERENCES channels(id) ON DELETE CASCADE,
                UNIQUE(user_id, channel_id)
            )
        `);

    // Table des commandes personnalisées
    this.db.exec(`
            CREATE TABLE IF NOT EXISTS custom_commands (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                channel_id INTEGER NOT NULL,
                command_name TEXT NOT NULL,
                response TEXT NOT NULL,
                created_by INTEGER,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                cooldown INTEGER DEFAULT 5,
                FOREIGN KEY (channel_id) REFERENCES channels(id) ON DELETE CASCADE,
                FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL,
                UNIQUE(channel_id, command_name)
            )
        `);

    console.log("Tables de base de données créées/vérifiées");
  }

  // Fermer la connexion à la base de données
  close() {
    if (this.db) {
      this.db.close();
      this.initialized = false;
      console.log("Connexion à la base de données fermée");
    }
  }

  // Préparer et mettre en cache une instruction SQL
  prepare(sql) {
    if (!this.initialized) this.init();
    return this.db.prepare(sql);
  }

  // Exécuter une requête et retourner tous les résultats
  query(sql, params = []) {
    const stmt = this.prepare(sql);
    return stmt.all(params);
  }

  // Exécuter une requête et retourner le premier résultat
  queryOne(sql, params = []) {
    const stmt = this.prepare(sql);
    return stmt.get(params);
  }

  // Exécuter une requête sans retourner de résultat
  execute(sql, params = []) {
    const stmt = this.prepare(sql);
    return stmt.run(params);
  }

  // Exécuter une série de requêtes dans une transaction
  transaction(callback) {
    if (!this.initialized) this.init();

    const transaction = this.db.transaction(callback);
    return transaction();
  }
}

// Créer et exporter une instance unique
const database = new Database();
module.exports = database;
```

### Module de gestion des utilisateurs

Créons un module pour gérer les utilisateurs dans la base de données :

```javascript
// src/utils/userManager.js
const database = require("./database");

class UserManager {
  // Obtenir ou créer un utilisateur
  getOrCreateUser(platform, userId, username) {
    // Vérifier si l'utilisateur existe déjà
    let user = database.queryOne(
      "SELECT * FROM users WHERE platform = ? AND user_id = ?",
      [platform, userId]
    );

    // Si l'utilisateur n'existe pas, le créer
    if (!user) {
      const result = database.execute(
        "INSERT INTO users (platform, user_id, username) VALUES (?, ?, ?)",
        [platform, userId, username]
      );

      user = {
        id: result.lastInsertRowid,
        platform,
        user_id: userId,
        username,
        created_at: new Date().toISOString(),
        last_seen: new Date().toISOString(),
      };
    } else {
      // Mettre à jour le nom d'utilisateur et last_seen si l'utilisateur existe déjà
      database.execute(
        "UPDATE users SET username = ?, last_seen = CURRENT_TIMESTAMP WHERE id = ?",
        [username, user.id]
      );
    }

    return user;
  }

  // Obtenir un utilisateur par son ID
  getUserById(userId) {
    return database.queryOne("SELECT * FROM users WHERE id = ?", [userId]);
  }

  // Obtenir un utilisateur par son ID de plateforme
  getUserByPlatformId(platform, platformUserId) {
    return database.queryOne(
      "SELECT * FROM users WHERE platform = ? AND user_id = ?",
      [platform, platformUserId]
    );
  }

  // Rechercher des utilisateurs
  searchUsers(query, platform = null, limit = 10) {
    const params = [`%${query}%`];
    let sql = "SELECT * FROM users WHERE username LIKE ?";

    if (platform) {
      sql += " AND platform = ?";
      params.push(platform);
    }

    sql += " ORDER BY last_seen DESC LIMIT ?";
    params.push(limit);

    return database.query(sql, params);
  }
}

// Créer et exporter une instance unique
const userManager = new UserManager();
module.exports = userManager;
```

### Module de gestion des canaux/serveurs

Créons un module pour gérer les serveurs Discord ou canaux Twitch :

```javascript
// src/utils/channelManager.js
const database = require("./database");

class ChannelManager {
  // Obtenir ou créer un canal
  getOrCreateChannel(platform, channelId, channelName, prefix = "!") {
    // Vérifier si le canal existe déjà
    let channel = database.queryOne(
      "SELECT * FROM channels WHERE platform = ? AND channel_id = ?",
      [platform, channelId]
    );

    // Si le canal n'existe pas, le créer
    if (!channel) {
      const result = database.execute(
        "INSERT INTO channels (platform, channel_id, channel_name, prefix) VALUES (?, ?, ?, ?)",
        [platform, channelId, channelName, prefix]
      );

      channel = {
        id: result.lastInsertRowid,
        platform,
        channel_id: channelId,
        channel_name: channelName,
        prefix,
        created_at: new Date().toISOString(),
      };
    } else {
      // Mettre à jour le nom du canal si nécessaire
      if (channel.channel_name !== channelName) {
        database.execute("UPDATE channels SET channel_name = ? WHERE id = ?", [
          channelName,
          channel.id,
        ]);
        channel.channel_name = channelName;
      }
    }

    return channel;
  }

  // Obtenir un canal par son ID
  getChannelById(channelId) {
    return database.queryOne("SELECT * FROM channels WHERE id = ?", [
      channelId,
    ]);
  }

  // Obtenir un canal par son ID de plateforme
  getChannelByPlatformId(platform, platformChannelId) {
    return database.queryOne(
      "SELECT * FROM channels WHERE platform = ? AND channel_id = ?",
      [platform, platformChannelId]
    );
  }

  // Mettre à jour le préfixe d'un canal
  updatePrefix(channelId, newPrefix) {
    database.execute("UPDATE channels SET prefix = ? WHERE id = ?", [
      newPrefix,
      channelId,
    ]);
  }

  // Obtenir tous les canaux d'une plateforme
  getChannelsByPlatform(platform) {
    return database.query("SELECT * FROM channels WHERE platform = ?", [
      platform,
    ]);
  }
}

// Créer et exporter une instance unique
const channelManager = new ChannelManager();
module.exports = channelManager;
```

### Module de gestion des points

Maintenant, améliorons notre système de points en utilisant la base de données :

```javascript
// src/utils/pointsManager.js
const database = require("./database");
const userManager = require("./userManager");
const channelManager = require("./channelManager");

class PointsManager {
  // Ajouter des points à un utilisateur
  addPoints(platform, channelId, userId, username, amount) {
    return database.transaction(() => {
      // Obtenir ou créer l'utilisateur et le canal
      const user = userManager.getOrCreateUser(platform, userId, username);
      const channel = channelManager.getOrCreateChannel(
        platform,
        channelId,
        channelId
      );

      // Vérifier si l'entrée de points existe déjà
      let pointsEntry = database.queryOne(
        "SELECT * FROM points WHERE user_id = ? AND channel_id = ?",
        [user.id, channel.id]
      );

      // Si l'entrée n'existe pas, la créer
      if (!pointsEntry) {
        database.execute(
          "INSERT INTO points (user_id, channel_id, points, last_gained) VALUES (?, ?, ?, CURRENT_TIMESTAMP)",
          [user.id, channel.id, amount]
        );

        return {
          userId: user.id,
          channelId: channel.id,
          points: amount,
          previousPoints: 0,
          newPoints: amount,
        };
      } else {
        // Mettre à jour les points existants
        const previousPoints = pointsEntry.points;
        const newPoints = previousPoints + amount;

        database.execute(
          "UPDATE points SET points = ?, last_gained = CURRENT_TIMESTAMP WHERE user_id = ? AND channel_id = ?",
          [newPoints, user.id, channel.id]
        );

        return {
          userId: user.id,
          channelId: channel.id,
          points: amount,
          previousPoints,
          newPoints,
        };
      }
    });
  }

  // Retirer des points à un utilisateur
  removePoints(platform, channelId, userId, username, amount) {
    return database.transaction(() => {
      // Obtenir ou créer l'utilisateur et le canal
      const user = userManager.getOrCreateUser(platform, userId, username);
      const channel = channelManager.getOrCreateChannel(
        platform,
        channelId,
        channelId
      );

      // Vérifier si l'entrée de points existe
      let pointsEntry = database.queryOne(
        "SELECT * FROM points WHERE user_id = ? AND channel_id = ?",
        [user.id, channel.id]
      );

      // Si l'entrée n'existe pas, retourner 0
      if (!pointsEntry) {
        return {
          userId: user.id,
          channelId: channel.id,
          points: 0,
          previousPoints: 0,
          newPoints: 0,
          success: false,
        };
      }

      // Calculer les nouveaux points (minimum 0)
      const previousPoints = pointsEntry.points;
      const pointsToRemove = Math.min(previousPoints, amount);
      const newPoints = previousPoints - pointsToRemove;

      // Mettre à jour les points
      database.execute(
        "UPDATE points SET points = ? WHERE user_id = ? AND channel_id = ?",
        [newPoints, user.id, channel.id]
      );

      return {
        userId: user.id,
        channelId: channel.id,
        points: pointsToRemove,
        previousPoints,
        newPoints,
        success: pointsToRemove > 0,
      };
    });
  }

  // Obtenir les points d'un utilisateur
  getPoints(platform, channelId, userId, username) {
    return database.transaction(() => {
      // Obtenir ou créer l'utilisateur et le canal
      const user = userManager.getOrCreateUser(platform, userId, username);
      const channel = channelManager.getOrCreateChannel(
        platform,
        channelId,
        channelId
      );

      // Obtenir les points
      const pointsEntry = database.queryOne(
        "SELECT * FROM points WHERE user_id = ? AND channel_id = ?",
        [user.id, channel.id]
      );

      return pointsEntry ? pointsEntry.points : 0;
    });
  }

  // Transférer des points entre utilisateurs
  transferPoints(
    platform,
    channelId,
    fromUserId,
    fromUsername,
    toUserId,
    toUsername,
    amount
  ) {
    return database.transaction(() => {
      // Vérifier que le montant est positif
      if (amount <= 0) {
        return { success: false, reason: "amount_invalid" };
      }

      // Vérifier que les utilisateurs sont différents
      if (fromUserId === toUserId) {
        return { success: false, reason: "same_user" };
      }

      // Obtenir les points actuels de l'expéditeur
      const currentPoints = this.getPoints(
        platform,
        channelId,
        fromUserId,
        fromUsername
      );

      // Vérifier si l'expéditeur a assez de points
      if (currentPoints < amount) {
        return {
          success: false,
          reason: "insufficient_points",
          currentPoints,
        };
      }

      // Retirer les points de l'expéditeur
      const removeResult = this.removePoints(
        platform,
        channelId,
        fromUserId,
        fromUsername,
        amount
      );

      // Ajouter les points au destinataire
      const addResult = this.addPoints(
        platform,
        channelId,
        toUserId,
        toUsername,
        amount
      );

      return {
        success: true,
        amount,
        fromUser: {
          id: fromUserId,
          username: fromUsername,
          newPoints: removeResult.newPoints,
        },
        toUser: {
          id: toUserId,
          username: toUsername,
          newPoints: addResult.newPoints,
        },
      };
    });
  }

  // Obtenir le classement des points
  getLeaderboard(platform, channelId, limit = 10) {
    // Obtenir le canal
    const channel = channelManager.getOrCreateChannel(
      platform,
      channelId,
      channelId
    );

    // Requête pour obtenir le classement
    return database.query(
      `
            SELECT u.username, p.points
            FROM points p
            JOIN users u ON p.user_id = u.id
            WHERE p.channel_id = ?
            ORDER BY p.points DESC
            LIMIT ?
        `,
      [channel.id, limit]
    );
  }

  // Récompenser les utilisateurs actifs (pour un système d'attribution automatique)
  rewardActiveUsers(platform, channelId, activeUsers, pointsAmount) {
    return database.transaction(() => {
      const results = [];

      for (const user of activeUsers) {
        const result = this.addPoints(
          platform,
          channelId,
          user.id,
          user.username,
          pointsAmount
        );
        results.push({
          userId: user.id,
          username: user.username,
          pointsAdded: pointsAmount,
          newTotal: result.newPoints,
        });
      }

      return results;
    });
  }
}

// Créer et exporter une instance unique
const pointsManager = new PointsManager();
module.exports = pointsManager;
```

### Module de gestion des commandes personnalisées

Améliorons notre système de commandes personnalisées avec la base de données :

```javascript
// src/utils/customCommandsManager.js
const database = require("./database");
const userManager = require("./userManager");
const channelManager = require("./channelManager");

class CustomCommandsManager {
  // Ajouter ou mettre à jour une commande
  addCommand(platform, channelId, commandName, response, createdBy) {
    return database.transaction(() => {
      // Obtenir ou créer le canal et l'utilisateur
      const channel = channelManager.getOrCreateChannel(
        platform,
        channelId,
        channelId
      );
      const user = userManager.getOrCreateUser(
        platform,
        createdBy.id,
        createdBy.username
      );

      // Vérifier si la commande existe déjà
      const existingCommand = database.queryOne(
        "SELECT * FROM custom_commands WHERE channel_id = ? AND command_name = ?",
        [channel.id, commandName]
      );

      if (existingCommand) {
        // Mettre à jour la commande existante
        database.execute(
          `
                    UPDATE custom_commands 
                    SET response = ?, updated_at = CURRENT_TIMESTAMP
                    WHERE id = ?
                `,
          [response, existingCommand.id]
        );

        return {
          id: existingCommand.id,
          commandName,
          response,
          isNew: false,
        };
      } else {
        // Créer une nouvelle commande
        const result = database.execute(
          `
                    INSERT INTO custom_commands (channel_id, command_name, response, created_by)
                    VALUES (?, ?, ?, ?)
                `,
          [channel.id, commandName, response, user.id]
        );

        return {
          id: result.lastInsertRowid,
          commandName,
          response,
          isNew: true,
        };
      }
    });
  }

  // Supprimer une commande
  removeCommand(platform, channelId, commandName) {
    return database.transaction(() => {
      // Obtenir le canal
      const channel = channelManager.getChannelByPlatformId(
        platform,
        channelId
      );

      if (!channel) return { success: false, reason: "channel_not_found" };

      // Vérifier si la commande existe
      const existingCommand = database.queryOne(
        "SELECT * FROM custom_commands WHERE channel_id = ? AND command_name = ?",
        [channel.id, commandName]
      );

      if (!existingCommand) {
        return { success: false, reason: "command_not_found" };
      }

      // Supprimer la commande
      database.execute("DELETE FROM custom_commands WHERE id = ?", [
        existingCommand.id,
      ]);

      return { success: true, commandName };
    });
  }

  // Obtenir une commande
  getCommand(platform, channelId, commandName) {
    // Obtenir le canal
    const channel = channelManager.getChannelByPlatformId(platform, channelId);

    if (!channel) return null;

    // Rechercher la commande
    return database.queryOne(
      `
            SELECT cc.*, u.username as creator_username
            FROM custom_commands cc
            LEFT JOIN users u ON cc.created_by = u.id
            WHERE cc.channel_id = ? AND cc.command_name = ?
        `,
      [channel.id, commandName]
    );
  }

  // Obtenir toutes les commandes d'un canal
  getAllCommands(platform, channelId) {
    // Obtenir le canal
    const channel = channelManager.getChannelByPlatformId(platform, channelId);

    if (!channel) return [];

    // Rechercher toutes les commandes
    return database.query(
      `
            SELECT cc.*, u.username as creator_username
            FROM custom_commands cc
            LEFT JOIN users u ON cc.created_by = u.id
            WHERE cc.channel_id = ?
            ORDER BY cc.command_name
        `,
      [channel.id]
    );
  }

  // Mettre à jour le cooldown d'une commande
  updateCooldown(platform, channelId, commandName, cooldown) {
    return database.transaction(() => {
      // Obtenir le canal
      const channel = channelManager.getChannelByPlatformId(
        platform,
        channelId
      );

      if (!channel) return { success: false, reason: "channel_not_found" };

      // Vérifier si la commande existe
      const existingCommand = database.queryOne(
        "SELECT * FROM custom_commands WHERE channel_id = ? AND command_name = ?",
        [channel.id, commandName]
      );

      if (!existingCommand) {
        return { success: false, reason: "command_not_found" };
      }

      // Mettre à jour le cooldown
      database.execute("UPDATE custom_commands SET cooldown = ? WHERE id = ?", [
        cooldown,
        existingCommand.id,
      ]);

      return {
        success: true,
        commandName,
        cooldown,
      };
    });
  }
}

// Créer et exporter une instance unique
const customCommandsManager = new CustomCommandsManager();
module.exports = customCommandsManager;
```

## Utilisation des modules de base de données dans nos bots

### Intégration avec le bot Discord

Modifions notre bot Discord pour utiliser notre nouvelle base de données :

#### Initialisation de la base de données (src/index.js pour Discord)

```javascript
// Ajoutez en haut du fichier
const database = require("./utils/database");

// Initialiser la base de données au démarrage
database.init();

// Assurez-vous de fermer proprement la connexion à la base de données
process.on("SIGINT", () => {
  console.log("Arrêt du bot...");
  database.close();
  process.exit(0);
});

process.on("SIGTERM", () => {
  console.log("Arrêt du bot...");
  database.close();
  process.exit(0);
});
```

#### Mise à jour de nos commandes Discord

Mettons à jour notre commande de points pour Discord :

```javascript
// src/commands/points.js (Discord)
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const pointsManager = require("../utils/pointsManager");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("points")
    .setDescription("Affiche vos points ou ceux d'un autre utilisateur")
    .addUserOption((option) =>
      option
        .setName("utilisateur")
        .setDescription("L'utilisateur dont vous voulez voir les points")
        .setRequired(false)
    ),

  async execute(interaction) {
    // Déterminer l'utilisateur cible
    const targetUser =
      interaction.options.getUser("utilisateur") || interaction.user;

    // Obtenir les points
    const points = await pointsManager.getPoints(
      "discord",
      interaction.guild.id,
      targetUser.id,
      targetUser.username
    );

    // Créer l'embed
    const pointsEmbed = new EmbedBuilder()
      .setColor(0x3498db)
      .setTitle(`Points de ${targetUser.username}`)
      .setDescription(
        `${targetUser} a **${points}** point${points !== 1 ? "s" : ""}.`
      )
      .setThumbnail(targetUser.displayAvatarURL({ dynamic: true }))
      .setFooter({ text: `ID: ${targetUser.id}` })
      .setTimestamp();

    await interaction.reply({ embeds: [pointsEmbed] });
  },
};
```

Mettons à jour notre commande de classement pour Discord :

```javascript
// src/commands/classement.js (Discord)
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const pointsManager = require("../utils/pointsManager");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("classement")
    .setDescription("Affiche le classement des membres par points")
    .addIntegerOption((option) =>
      option
        .setName("limite")
        .setDescription("Nombre de membres à afficher (par défaut: 10)")
        .setMinValue(1)
        .setMaxValue(25)
        .setRequired(false)
    ),

  async execute(interaction) {
    // Obtenir la limite
    const limit = interaction.options.getInteger("limite") || 10;

    // Récupérer le classement
    const leaderboard = await pointsManager.getLeaderboard(
      "discord",
      interaction.guild.id,
      limit
    );

    if (leaderboard.length === 0) {
      return interaction.reply({
        content: "Aucun membre n'a encore gagné de points sur ce serveur.",
        ephemeral: true,
      });
    }

    // Créer la description du classement
    let description = "";

    leaderboard.forEach((entry, index) => {
      // Ajouter des médailles pour les 3 premiers
      let medal = "";
      if (index === 0) medal = "🥇 ";
      else if (index === 1) medal = "🥈 ";
      else if (index === 2) medal = "🥉 ";

      description += `${medal}**#${index + 1}** ${entry.username} - ${
        entry.points
      } point${entry.points !== 1 ? "s" : ""}\n`;
    });

    // Créer l'embed
    const leaderboardEmbed = new EmbedBuilder()
      .setColor(0xf1c40f)
      .setTitle(`🏆 Classement des points - Top ${limit}`)
      .setDescription(description)
      .setFooter({ text: "Continuez à discuter pour gagner des points !" })
      .setTimestamp();

    await interaction.reply({ embeds: [leaderboardEmbed] });
  },
};
```

### Intégration avec le bot Twitch

Mettons à jour notre bot Twitch pour utiliser la base de données :

#### Initialisation de la base de données (src/index.js pour Twitch)

```javascript
// Ajoutez en haut du fichier
const database = require("./utils/database");

// Initialiser la base de données au démarrage
database.init();

// Assurez-vous de fermer proprement la connexion à la base de données
process.on("SIGINT", () => {
  console.log("Arrêt du bot...");
  database.close();
  process.exit(0);
});

process.on("SIGTERM", () => {
  console.log("Arrêt du bot...");
  database.close();
  process.exit(0);
});
```

#### Mise à jour de nos commandes Twitch

Mettons à jour notre commande de points pour Twitch :

```javascript
// src/commands/points.js (Twitch)
const pointsManager = require("../utils/pointsManager");

module.exports = {
  name: "points",
  description: "Affiche vos points ou ceux d'un autre utilisateur",
  execute: async (client, channel, tags, args) => {
    // Déterminer l'utilisateur cible
    const targetUsername =
      args.length > 0 ? args[0].toLowerCase().replace("@", "") : tags.username;

    // ID du canal Twitch (sans le #)
    const channelId = channel.slice(1);

    // Obtenir les points
    const points = await pointsManager.getPoints(
      "twitch",
      channelId,
      targetUsername === tags.username ? tags["user-id"] : targetUsername,
      targetUsername
    );

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

Mettons à jour notre commande de classement pour Twitch :

```javascript
// src/commands/classement.js (Twitch)
const pointsManager = require("../utils/pointsManager");

module.exports = {
  name: "classement",
  description: "Affiche le classement des utilisateurs par points",
  execute: async (client, channel, tags, args) => {
    // Obtenir le classement (top 5 par défaut)
    const limit = args.length > 0 && !isNaN(args[0]) ? parseInt(args[0]) : 5;

    // ID du canal Twitch (sans le #)
    const channelId = channel.slice(1);

    // Récupérer le classement
    const leaderboard = await pointsManager.getLeaderboard(
      "twitch",
      channelId,
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

## MongoDB : Une alternative NoSQL

Si vous préférez une approche NoSQL plus flexible, MongoDB est une excellente option, particulièrement pour les bots qui doivent gérer des données complexes ou qui sont déployés à grande échelle.

### Installation de MongoDB avec Node.js

```bash
npm install mongodb mongoose
```

Nous utiliserons Mongoose, qui est un ODM (Object-Document Mapper) pour MongoDB, facilitant la modélisation et les requêtes.

### Configuration de Mongoose

```javascript
// src/utils/mongoose.js
const mongoose = require("mongoose");
require("dotenv").config();

// URL de connexion à MongoDB
const MONGODB_URI =
  process.env.MONGODB_URI || "mongodb://localhost:27017/botdb";

// Configuration de Mongoose
mongoose.set("strictQuery", false);

// Fonction de connexion
async function connectToDatabase() {
  try {
    await mongoose.connect(MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });

    console.log("Connecté à MongoDB avec succès");
    return true;
  } catch (error) {
    console.error("Erreur de connexion à MongoDB:", error);
    return false;
  }
}

// Fonction de déconnexion
async function disconnectFromDatabase() {
  try {
    await mongoose.disconnect();
    console.log("Déconnecté de MongoDB avec succès");
    return true;
  } catch (error) {
    console.error("Erreur lors de la déconnexion de MongoDB:", error);
    return false;
  }
}

module.exports = {
  connectToDatabase,
  disconnectFromDatabase,
  mongoose,
};
```

### Modèles Mongoose

Créons quelques modèles pour notre base de données MongoDB :

```javascript
// src/models/User.js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema(
  {
    platform: {
      type: String,
      required: true,
      enum: ["discord", "twitch"],
    },
    platformId: {
      type: String,
      required: true,
    },
    username: {
      type: String,
      required: true,
    },
    avatarUrl: String,
    createdAt: {
      type: Date,
      default: Date.now,
    },
    lastSeen: {
      type: Date,
      default: Date.now,
    },
  },
  {
    timestamps: true,
  }
);

// Index composite pour rechercher rapidement un utilisateur par plateforme et ID
userSchema.index({ platform: 1, platformId: 1 }, { unique: true });

module.exports = mongoose.model("User", userSchema);
```

```javascript
// src/models/Channel.js
const mongoose = require("mongoose");

const channelSchema = new mongoose.Schema(
  {
    platform: {
      type: String,
      required: true,
      enum: ["discord", "twitch"],
    },
    platformId: {
      type: String,
      required: true,
    },
    name: {
      type: String,
      required: true,
    },
    prefix: {
      type: String,
      default: "!",
    },
    settings: {
      welcomeMessage: String,
      autoMod: {
        enabled: { type: Boolean, default: false },
        filterLinks: { type: Boolean, default: false },
        filterSpam: { type: Boolean, default: false },
      },
      pointsSystem: {
        enabled: { type: Boolean, default: true },
        pointsPerMessage: { type: Number, default: 1 },
        pointsInterval: { type: Number, default: 10 }, // Minutes
      },
    },
  },
  {
    timestamps: true,
  }
);

// Index composite pour rechercher rapidement un canal par plateforme et ID
channelSchema.index({ platform: 1, platformId: 1 }, { unique: true });

module.exports = mongoose.model("Channel", channelSchema);
```

```javascript
// src/models/Points.js
const mongoose = require("mongoose");

const pointsSchema = new mongoose.Schema(
  {
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true,
    },
    channel: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Channel",
      required: true,
    },
    points: {
      type: Number,
      default: 0,
      min: 0,
    },
    lastGained: {
      type: Date,
      default: Date.now,
    },
    history: [
      {
        amount: Number,
        reason: String,
        timestamp: {
          type: Date,
          default: Date.now,
        },
      },
    ],
  },
  {
    timestamps: true,
  }
);

// Index composite pour rechercher rapidement les points d'un utilisateur dans un canal
pointsSchema.index({ user: 1, channel: 1 }, { unique: true });

module.exports = mongoose.model("Points", pointsSchema);
```

```javascript
// src/models/CustomCommand.js
const mongoose = require("mongoose");

const customCommandSchema = new mongoose.Schema(
  {
    channel: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Channel",
      required: true,
    },
    name: {
      type: String,
      required: true,
    },
    response: {
      type: String,
      required: true,
    },
    createdBy: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
    },
    cooldown: {
      type: Number,
      default: 5,
      min: 0,
    },
    usageCount: {
      type: Number,
      default: 0,
    },
    lastUsed: Date,
  },
  {
    timestamps: true,
  }
);

// Index composite pour rechercher rapidement une commande par canal et nom
customCommandSchema.index({ channel: 1, name: 1 }, { unique: true });

module.exports = mongoose.model("CustomCommand", customCommandSchema);
```

### Services pour interagir avec MongoDB

Créons maintenant des services pour interagir avec nos modèles MongoDB :

```javascript
// src/services/userService.js
const User = require("../models/User");

class UserService {
  // Obtenir ou créer un utilisateur
  async getOrCreateUser(platform, platformId, username, avatarUrl = null) {
    try {
      // Rechercher l'utilisateur
      let user = await User.findOne({ platform, platformId });

      if (user) {
        // Mettre à jour les informations de l'utilisateur
        user.username = username;
        user.lastSeen = new Date();

        if (avatarUrl) {
          user.avatarUrl = avatarUrl;
        }

        await user.save();
        return user;
      }

      // Créer un nouvel utilisateur
      user = new User({
        platform,
        platformId,
        username,
        avatarUrl,
        lastSeen: new Date(),
      });

      await user.save();
      return user;
    } catch (error) {
      console.error("Erreur dans getOrCreateUser:", error);
      throw error;
    }
  }

  // Obtenir un utilisateur par son ID
  async getUserById(userId) {
    try {
      return await User.findById(userId);
    } catch (error) {
      console.error("Erreur dans getUserById:", error);
      throw error;
    }
  }

  // Obtenir un utilisateur par son ID de plateforme
  async getUserByPlatformId(platform, platformId) {
    try {
      return await User.findOne({ platform, platformId });
    } catch (error) {
      console.error("Erreur dans getUserByPlatformId:", error);
      throw error;
    }
  }

  // Rechercher des utilisateurs
  async searchUsers(query, platform = null, limit = 10) {
    try {
      const searchQuery = { username: new RegExp(query, "i") };

      if (platform) {
        searchQuery.platform = platform;
      }

      return await User.find(searchQuery).sort({ lastSeen: -1 }).limit(limit);
    } catch (error) {
      console.error("Erreur dans searchUsers:", error);
      throw error;
    }
  }
}

module.exports = new UserService();
```

```javascript
// src/services/channelService.js
const Channel = require("../models/Channel");

class ChannelService {
  // Obtenir ou créer un canal
  async getOrCreateChannel(platform, platformId, name, prefix = "!") {
    try {
      // Rechercher le canal
      let channel = await Channel.findOne({ platform, platformId });

      if (channel) {
        // Mettre à jour les informations du canal si nécessaire
        if (channel.name !== name) {
          channel.name = name;
          await channel.save();
        }

        return channel;
      }

      // Créer un nouveau canal
      channel = new Channel({
        platform,
        platformId,
        name,
        prefix,
      });

      await channel.save();
      return channel;
    } catch (error) {
      console.error("Erreur dans getOrCreateChannel:", error);
      throw error;
    }
  }

  // Obtenir un canal par son ID
  async getChannelById(channelId) {
    try {
      return await Channel.findById(channelId);
    } catch (error) {
      console.error("Erreur dans getChannelById:", error);
      throw error;
    }
  }

  // Obtenir un canal par son ID de plateforme
  async getChannelByPlatformId(platform, platformId) {
    try {
      return await Channel.findOne({ platform, platformId });
    } catch (error) {
      console.error("Erreur dans getChannelByPlatformId:", error);
      throw error;
    }
  }

  // Mettre à jour le préfixe d'un canal
  async updatePrefix(channelId, newPrefix) {
    try {
      await Channel.findByIdAndUpdate(channelId, { prefix: newPrefix });
      return true;
    } catch (error) {
      console.error("Erreur dans updatePrefix:", error);
      throw error;
    }
  }

  // Mettre à jour les paramètres d'un canal
  async updateSettings(channelId, settings) {
    try {
      await Channel.findByIdAndUpdate(channelId, {
        $set: { settings },
      });
      return true;
    } catch (error) {
      console.error("Erreur dans updateSettings:", error);
      throw error;
    }
  }

  // Obtenir tous les canaux d'une plateforme
  async getChannelsByPlatform(platform) {
    try {
      return await Channel.find({ platform });
    } catch (error) {
      console.error("Erreur dans getChannelsByPlatform:", error);
      throw error;
    }
  }
}

module.exports = new ChannelService();
```

```javascript
// src/services/pointsService.js
const Points = require("../models/Points");
const userService = require("./userService");
const channelService = require("./channelService");

class PointsService {
  // Ajouter des points à un utilisateur
  async addPoints(
    platform,
    channelPlatformId,
    userPlatformId,
    username,
    amount,
    reason = "add"
  ) {
    try {
      // Obtenir ou créer l'utilisateur et le canal
      const user = await userService.getOrCreateUser(
        platform,
        userPlatformId,
        username
      );
      const channel = await channelService.getOrCreateChannel(
        platform,
        channelPlatformId,
        channelPlatformId
      );

      // Rechercher l'entrée de points
      let pointsEntry = await Points.findOne({
        user: user._id,
        channel: channel._id,
      });

      if (!pointsEntry) {
        // Créer une nouvelle entrée de points
        pointsEntry = new Points({
          user: user._id,
          channel: channel._id,
          points: amount,
          lastGained: new Date(),
          history: [
            {
              amount,
              reason,
              timestamp: new Date(),
            },
          ],
        });

        await pointsEntry.save();

        return {
          userId: user._id,
          channelId: channel._id,
          points: amount,
          previousPoints: 0,
          newPoints: amount,
        };
      } else {
        // Mettre à jour les points existants
        const previousPoints = pointsEntry.points;
        const newPoints = previousPoints + amount;

        pointsEntry.points = newPoints;
        pointsEntry.lastGained = new Date();

        // Ajouter à l'historique
        pointsEntry.history.push({
          amount,
          reason,
          timestamp: new Date(),
        });

        // Limiter la taille de l'historique à 100 entrées
        if (pointsEntry.history.length > 100) {
          pointsEntry.history = pointsEntry.history.slice(-100);
        }

        await pointsEntry.save();

        return {
          userId: user._id,
          channelId: channel._id,
          points: amount,
          previousPoints,
          newPoints,
        };
      }
    } catch (error) {
      console.error("Erreur dans addPoints:", error);
      throw error;
    }
  }

  // Retirer des points à un utilisateur
  async removePoints(
    platform,
    channelPlatformId,
    userPlatformId,
    username,
    amount,
    reason = "remove"
  ) {
    try {
      // Obtenir ou créer l'utilisateur et le canal
      const user = await userService.getOrCreateUser(
        platform,
        userPlatformId,
        username
      );
      const channel = await channelService.getOrCreateChannel(
        platform,
        channelPlatformId,
        channelPlatformId
      );

      // Rechercher l'entrée de points
      let pointsEntry = await Points.findOne({
        user: user._id,
        channel: channel._id,
      });

      if (!pointsEntry) {
        return {
          userId: user._id,
          channelId: channel._id,
          points: 0,
          previousPoints: 0,
          newPoints: 0,
          success: false,
        };
      }

      // Calculer les nouveaux points (minimum 0)
      const previousPoints = pointsEntry.points;
      const pointsToRemove = Math.min(previousPoints, amount);
      const newPoints = previousPoints - pointsToRemove;

      // Mettre à jour les points
      pointsEntry.points = newPoints;

      // Ajouter à l'historique seulement si des points ont été retirés
      if (pointsToRemove > 0) {
        pointsEntry.history.push({
          amount: -pointsToRemove,
          reason,
          timestamp: new Date(),
        });

        // Limiter la taille de l'historique à 100 entrées
        if (pointsEntry.history.length > 100) {
          pointsEntry.history = pointsEntry.history.slice(-100);
        }
      }

      await pointsEntry.save();

      return {
        userId: user._id,
        channelId: channel._id,
        points: pointsToRemove,
        previousPoints,
        newPoints,
        success: pointsToRemove > 0,
      };
    } catch (error) {
      console.error("Erreur dans removePoints:", error);
      throw error;
    }
  }

  // Obtenir les points d'un utilisateur
  async getPoints(platform, channelPlatformId, userPlatformId, username) {
    try {
      // Obtenir ou créer l'utilisateur et le canal
      const user = await userService.getOrCreateUser(
        platform,
        userPlatformId,
        username
      );
      const channel = await channelService.getOrCreateChannel(
        platform,
        channelPlatformId,
        channelPlatformId
      );

      // Rechercher l'entrée de points
      const pointsEntry = await Points.findOne({
        user: user._id,
        channel: channel._id,
      });

      return pointsEntry ? pointsEntry.points : 0;
    } catch (error) {
      console.error("Erreur dans getPoints:", error);
      throw error;
    }
  }

  // Obtenir le classement des points
  async getLeaderboard(platform, channelPlatformId, limit = 10) {
    try {
      // Obtenir le canal
      const channel = await channelService.getOrCreateChannel(
        platform,
        channelPlatformId,
        channelPlatformId
      );

      // Rechercher les entrées de points pour ce canal
      const leaderboard = await Points.find({ channel: channel._id })
        .sort({ points: -1 })
        .limit(limit)
        .populate("user", "username");

      // Formater les résultats
      return leaderboard.map((entry) => ({
        username: entry.user.username,
        points: entry.points,
      }));
    } catch (error) {
      console.error("Erreur dans getLeaderboard:", error);
      throw error;
    }
  }
}

module.exports = new PointsService();
```

```javascript
// src/services/customCommandService.js
const CustomCommand = require("../models/CustomCommand");
const userService = require("./userService");
const channelService = require("./channelService");

class CustomCommandService {
  // Ajouter ou mettre à jour une commande
  async addCommand(
    platform,
    channelPlatformId,
    commandName,
    response,
    createdBy
  ) {
    try {
      // Obtenir ou créer le canal et l'utilisateur
      const channel = await channelService.getOrCreateChannel(
        platform,
        channelPlatformId,
        channelPlatformId
      );
      const user = await userService.getOrCreateUser(
        platform,
        createdBy.id,
        createdBy.username
      );

      // Rechercher la commande existante
      let command = await CustomCommand.findOne({
        channel: channel._id,
        name: commandName,
      });

      if (command) {
        // Mettre à jour la commande existante
        command.response = response;
        await command.save();

        return {
          id: command._id,
          commandName,
          response,
          isNew: false,
        };
      } else {
        // Créer une nouvelle commande
        command = new CustomCommand({
          channel: channel._id,
          name: commandName,
          response,
          createdBy: user._id,
        });

        await command.save();

        return {
          id: command._id,
          commandName,
          response,
          isNew: true,
        };
      }
    } catch (error) {
      console.error("Erreur dans addCommand:", error);
      throw error;
    }
  }

  // Supprimer une commande
  async removeCommand(platform, channelPlatformId, commandName) {
    try {
      // Obtenir le canal
      const channel = await channelService.getChannelByPlatformId(
        platform,
        channelPlatformId
      );

      if (!channel) return { success: false, reason: "channel_not_found" };

      // Rechercher et supprimer la commande
      const result = await CustomCommand.deleteOne({
        channel: channel._id,
        name: commandName,
      });

      return {
        success: result.deletedCount > 0,
        commandName,
        reason: result.deletedCount > 0 ? null : "command_not_found",
      };
    } catch (error) {
      console.error("Erreur dans removeCommand:", error);
      throw error;
    }
  }

  // Obtenir une commande
  async getCommand(platform, channelPlatformId, commandName) {
    try {
      // Obtenir le canal
      const channel = await channelService.getChannelByPlatformId(
        platform,
        channelPlatformId
      );

      if (!channel) return null;

      // Rechercher la commande
      return await CustomCommand.findOne({
        channel: channel._id,
        name: commandName,
      }).populate("createdBy", "username");
    } catch (error) {
      console.error("Erreur dans getCommand:", error);
      throw error;
    }
  }

  // Obtenir toutes les commandes d'un canal
  async getAllCommands(platform, channelPlatformId) {
    try {
      // Obtenir le canal
      const channel = await channelService.getChannelByPlatformId(
        platform,
        channelPlatformId
      );

      if (!channel) return [];

      // Rechercher toutes les commandes
      return await CustomCommand.find({
        channel: channel._id,
      })
        .populate("createdBy", "username")
        .sort({ name: 1 });
    } catch (error) {
      console.error("Erreur dans getAllCommands:", error);
      throw error;
    }
  }

  // Mettre à jour le compteur d'utilisation d'une commande
  async trackCommandUsage(commandId) {
    try {
      await CustomCommand.findByIdAndUpdate(commandId, {
        $inc: { usageCount: 1 },
        lastUsed: new Date(),
      });
    } catch (error) {
      console.error("Erreur dans trackCommandUsage:", error);
      // Ne pas faire échouer l'exécution de la commande pour cette erreur
    }
  }
}

module.exports = new CustomCommandService();
```

### Initialisation de MongoDB dans nos bots

Pour utiliser MongoDB dans nos bots, nous devons l'initialiser au démarrage :

```javascript
// Ajoutez en haut de src/index.js (pour Discord ou Twitch)
const {
  connectToDatabase,
  disconnectFromDatabase,
} = require("./utils/mongoose");

// Connexion à la base de données au démarrage
(async () => {
  try {
    await connectToDatabase();
    // Continuer l'initialisation du bot seulement si la connexion à la DB est établie
    initializeBot();
  } catch (error) {
    console.error(
      "Erreur fatale lors de la connexion à la base de données:",
      error
    );
    process.exit(1);
  }
})();

// Assurez-vous de fermer proprement la connexion à la base de données
process.on("SIGINT", async () => {
  console.log("Arrêt du bot...");
  await disconnectFromDatabase();
  process.exit(0);
});

process.on("SIGTERM", async () => {
  console.log("Arrêt du bot...");
  await disconnectFromDatabase();
  process.exit(0);
});

// Fonction pour initialiser le bot
function initializeBot() {
  // Le reste de votre code d'initialisation du bot ici
}
```

## Avantages et inconvénients des différentes bases de données

### SQLite

**Avantages :**

- Simple à configurer (pas de serveur distinct)
- Fichier unique, facile à sauvegarder
- Bonne performance pour les petites applications
- SQL standard pour les requêtes

**Inconvénients :**

- Pas adapté aux applications à grande échelle
- Verrous au niveau du fichier (problèmes potentiels de concurrence)
- Fonctionnalités limitées par rapport aux SGBD complets

### MongoDB

**Avantages :**

- Schéma flexible, excellent pour les données variables
- Excellente scalabilité horizontale
- Bonne performance pour les lectures/écritures
- Format JSON naturel pour JavaScript

**Inconvénients :**

- Configuration plus complexe
- Nécessite plus de ressources
- Transactions complexes moins efficaces qu'en SQL

### MySQL/PostgreSQL

**Avantages :**

- Très robustes et fiables
- Excellentes performances pour les requêtes complexes
- Support des transactions ACID
- Outils d'administration matures

**Inconvénients :**

- Nécessite un serveur séparé
- Configuration initiale plus complexe
- Schéma rigide (moins flexible que NoSQL)

## Choisir la bonne base de données

Voici quelques conseils pour choisir la base de données la plus adaptée à votre bot :

1. **SQLite** : Idéal pour les petits bots personnels ou utilisés sur quelques serveurs.

   - Parfait pour débuter
   - Facile à déployer
   - Bonne solution jusqu'à environ 1000 utilisateurs

2. **MongoDB** : Excellent pour les bots de taille moyenne avec des données variées.

   - Adapté pour les bots avec des fonctionnalités évolutives
   - Bon choix si vous prévoyez d'ajouter souvent de nouvelles fonctionnalités
   - Pratique pour les bots utilisés sur de nombreux serveurs (10-1000+)

3. **MySQL/PostgreSQL** : Recommandés pour les grands bots avec des données relationnelles.
   - Idéals pour les bots publics à grande échelle
   - Parfaits si vous avez beaucoup de données interdépendantes
   - Bons pour les bots nécessitant des analyses complexes

## Conclusion

Dans ce chapitre, nous avons exploré comment intégrer différents types de bases de données à vos bots Discord et Twitch :

- Nous avons commencé par SQLite, une solution simple mais puissante pour les petits à moyens bots
- Nous avons créé plusieurs modules pour gérer les utilisateurs, les canaux, les points et les commandes personnalisées
- Nous avons exploré MongoDB comme alternative NoSQL plus flexible
- Nous avons vu comment choisir la base de données la plus adaptée à vos besoins

L'utilisation d'une base de données permettra à votre bot de stocker des données de manière persistante, de gérer efficacement de grandes quantités d'informations et d'offrir des fonctionnalités plus avancées à vos utilisateurs.

Dans le prochain chapitre, nous explorerons des fonctionnalités avancées pour vos bots Discord et Twitch.

## Ressources supplémentaires

- [Documentation SQLite](https://www.sqlite.org/docs.html)
- [Documentation Better-SQLite3](https://github.com/JoshuaWise/better-sqlite3/blob/master/docs/api.md)
- [Documentation MongoDB](https://docs.mongodb.com/)
- [Documentation Mongoose](https://mongoosejs.com/docs/guide.html)
- [Documentation MySQL](https://dev.mysql.com/doc/)
- [Documentation PostgreSQL](https://www.postgresql.org/docs/)
