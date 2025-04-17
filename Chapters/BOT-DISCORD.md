# Chapitre 4 - Création d'un Bot Discord

Dans ce chapitre, nous allons apprendre à créer un bot Discord fonctionnel en utilisant discord.js, la bibliothèque Node.js la plus populaire pour interagir avec l'API Discord.

## Préparation et configuration

### Création d'une application Discord

Avant de commencer à coder, vous devez créer une application Discord et obtenir un token d'authentification :

1. Connectez-vous au [Portail des développeurs Discord](https://discord.com/developers/applications)
2. Cliquez sur "New Application" en haut à droite
3. Donnez un nom à votre application et cliquez sur "Create"
4. Dans le menu latéral, cliquez sur "Bot"
5. Cliquez sur "Add Bot" puis confirmez
6. Sous la section "TOKEN", cliquez sur "Copy" pour copier votre token (gardez-le secret !)
7. Activez les "Privileged Gateway Intents" dont vous aurez besoin :
   - MESSAGE CONTENT INTENT (pour lire le contenu des messages)
   - SERVER MEMBERS INTENT (pour accéder aux membres du serveur)
   - PRESENCE INTENT (pour suivre la présence des utilisateurs)

### Configuration des permissions et invitation

Pour inviter votre bot sur un serveur :

1. Dans le menu latéral, cliquez sur "OAuth2" puis "URL Generator"
2. Dans "SCOPES", cochez "bot" et "applications.commands"
3. Dans "BOT PERMISSIONS", sélectionnez les permissions nécessaires (commencez par "Send Messages", "Read Messages/View Channels", "Embed Links")
4. Copiez l'URL générée et ouvrez-la dans votre navigateur
5. Sélectionnez le serveur où vous souhaitez ajouter votre bot et confirmez

## Installation et configuration du projet

Créons maintenant la structure de base de notre bot :

1. Créez un nouveau dossier pour votre projet :

```bash
mkdir mon-bot-discord
cd mon-bot-discord
```

2. Initialisez un projet Node.js :

```bash
npm init -y
```

3. Installez discord.js et dotenv (pour gérer les variables d'environnement) :

```bash
npm install discord.js dotenv
```

4. Créez un fichier `.env` pour stocker votre token en toute sécurité :

```
TOKEN=votre_token_discord_ici
```

5. Créez un fichier `.gitignore` (si vous utilisez Git) :

```
node_modules/
.env
```

6. Créez la structure de dossiers suivante :

```
mon-bot-discord/
│
├── node_modules/
├── src/
│   ├── commands/    # Dossier pour les commandes
│   ├── events/      # Dossier pour les gestionnaires d'événements
│   ├── utils/       # Dossier pour les fonctions utilitaires
│   └── index.js     # Point d'entrée principal
│
├── .env             # Variables d'environnement
├── .gitignore
└── package.json
```

## Premier bot Discord fonctionnel

Commençons par créer un bot simple qui répond à des commandes de base :

### 1. Fichier principal (src/index.js)

```javascript
// src/index.js
require("dotenv").config(); // Charger les variables d'environnement
const { Client, GatewayIntentBits, Collection } = require("discord.js");
const fs = require("fs");
const path = require("path");

// Créer un nouveau client Discord
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers,
  ],
});

// Collection pour stocker les commandes
client.commands = new Collection();

// Chargement des commandes
const commandsPath = path.join(__dirname, "commands");
const commandFiles = fs
  .readdirSync(commandsPath)
  .filter((file) => file.endsWith(".js"));

for (const file of commandFiles) {
  const filePath = path.join(commandsPath, file);
  const command = require(filePath);

  // Vérifier que la commande a bien les propriétés nécessaires
  if ("data" in command && "execute" in command) {
    client.commands.set(command.data.name, command);
    console.log(`✓ Commande chargée : ${command.data.name}`);
  } else {
    console.warn(
      `⚠ La commande ${file} manque de propriétés 'data' ou 'execute'`
    );
  }
}

// Chargement des événements
const eventsPath = path.join(__dirname, "events");
const eventFiles = fs
  .readdirSync(eventsPath)
  .filter((file) => file.endsWith(".js"));

for (const file of eventFiles) {
  const filePath = path.join(eventsPath, file);
  const event = require(filePath);

  if (event.once) {
    client.once(event.name, (...args) => event.execute(...args));
  } else {
    client.on(event.name, (...args) => event.execute(...args, client));
  }

  console.log(`✓ Événement chargé : ${event.name}`);
}

// Connexion à Discord
client.login(process.env.TOKEN);
```

### 2. Événements (src/events/)

#### Événement Ready (src/events/ready.js)

```javascript
// src/events/ready.js
module.exports = {
  name: "ready",
  once: true,
  execute(client) {
    console.log(`✅ Bot connecté en tant que ${client.user.tag} !`);

    // Définir le statut et l'activité du bot
    client.user.setPresence({
      activities: [{ name: "/aide pour voir les commandes", type: 3 }], // Type 3 = "Watching"
      status: "online",
    });
  },
};
```

#### Événement InteractionCreate (src/events/interactionCreate.js)

```javascript
// src/events/interactionCreate.js
module.exports = {
  name: "interactionCreate",
  once: false,
  async execute(interaction, client) {
    // Ignorer les interactions qui ne sont pas des commandes
    if (!interaction.isCommand()) return;

    // Récupérer la commande
    const command = client.commands.get(interaction.commandName);

    // Si la commande n'existe pas, on ignore
    if (!command) return;

    try {
      // Exécuter la commande
      await command.execute(interaction);
    } catch (error) {
      console.error(
        `Erreur lors de l'exécution de la commande ${interaction.commandName}:`,
        error
      );

      // Répondre à l'utilisateur qu'il y a eu une erreur
      const replyContent = {
        content:
          "Une erreur s'est produite lors de l'exécution de cette commande.",
        ephemeral: true, // Message visible uniquement par l'utilisateur qui a utilisé la commande
      };

      if (interaction.replied || interaction.deferred) {
        await interaction.followUp(replyContent);
      } else {
        await interaction.reply(replyContent);
      }
    }
  },
};
```

#### Événement MessageCreate (src/events/messageCreate.js)

```javascript
// src/events/messageCreate.js
module.exports = {
  name: "messageCreate",
  once: false,
  async execute(message, client) {
    // Ignorer les messages du bot lui-même ou des autres bots
    if (message.author.bot) return;

    // Simple exemple de réponse à un message contenant un mot-clé
    if (message.content.toLowerCase().includes("bonjour")) {
      await message.reply(
        `Bonjour ${message.author.username} ! Comment vas-tu ?`
      );
    }

    // Vous pouvez également implémenter un système de commandes basé sur un préfixe ici
    // Exemple : !aide, !ping, etc.
  },
};
```

### 3. Commandes (src/commands/)

#### Commande Ping (src/commands/ping.js)

```javascript
// src/commands/ping.js
const { SlashCommandBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("ping")
    .setDescription("Répond avec Pong et affiche la latence du bot"),

  async execute(interaction) {
    // Mesurer le temps de réponse
    const sent = await interaction.reply({
      content: "Calcul de la latence...",
      fetchReply: true,
    });
    const latence = sent.createdTimestamp - interaction.createdTimestamp;
    const apiLatence = Math.round(interaction.client.ws.ping);

    await interaction.editReply(
      `Pong ! 🏓\n- Latence : ${latence}ms\n- Latence API : ${apiLatence}ms`
    );
  },
};
```

#### Commande Aide (src/commands/aide.js)

```javascript
// src/commands/aide.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("aide")
    .setDescription("Affiche la liste des commandes disponibles"),

  async execute(interaction) {
    // Créer un embed pour afficher l'aide de manière plus jolie
    const helpEmbed = new EmbedBuilder()
      .setColor(0x0099ff)
      .setTitle("📚 Aide - Liste des commandes")
      .setDescription("Voici la liste des commandes disponibles :")
      .addFields(
        { name: "/aide", value: "Affiche cette liste de commandes" },
        { name: "/ping", value: "Vérifie la latence du bot" },
        {
          name: "/info",
          value: "Affiche des informations sur le serveur ou un utilisateur",
        },
        { name: "/roll", value: "Lance un dé et donne un résultat aléatoire" }
      )
      .setFooter({
        text: "Bot créé avec Discord.js",
        iconURL: interaction.client.user.displayAvatarURL(),
      })
      .setTimestamp();

    // Envoyer l'embed
    await interaction.reply({ embeds: [helpEmbed] });
  },
};
```

#### Commande Info (src/commands/info.js)

```javascript
// src/commands/info.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("info")
    .setDescription("Obtenir des informations sur le serveur ou un utilisateur")
    .addSubcommand((subcommand) =>
      subcommand
        .setName("serveur")
        .setDescription("Informations sur le serveur")
    )
    .addSubcommand((subcommand) =>
      subcommand
        .setName("utilisateur")
        .setDescription("Informations sur un utilisateur")
        .addUserOption((option) =>
          option
            .setName("cible")
            .setDescription(
              "L'utilisateur dont vous voulez voir les informations"
            )
            .setRequired(true)
        )
    ),

  async execute(interaction) {
    const subcommand = interaction.options.getSubcommand();

    if (subcommand === "serveur") {
      const guild = interaction.guild;

      // Créer un embed avec les informations du serveur
      const serverEmbed = new EmbedBuilder()
        .setColor(0x0099ff)
        .setTitle(`📊 Informations sur ${guild.name}`)
        .setThumbnail(guild.iconURL({ dynamic: true }))
        .addFields(
          { name: "ID", value: guild.id, inline: true },
          {
            name: "Créé le",
            value: `<t:${Math.floor(guild.createdTimestamp / 1000)}:R>`,
            inline: true,
          },
          { name: "Propriétaire", value: `<@${guild.ownerId}>`, inline: true },
          { name: "Membres", value: `${guild.memberCount}`, inline: true },
          {
            name: "Salons",
            value: `${guild.channels.cache.size}`,
            inline: true,
          },
          { name: "Rôles", value: `${guild.roles.cache.size}`, inline: true },
          {
            name: "Boosts",
            value: `${guild.premiumSubscriptionCount || 0}`,
            inline: true,
          },
          {
            name: "Niveau de boost",
            value: `${guild.premiumTier || 0}`,
            inline: true,
          }
        )
        .setFooter({ text: "Demandé par " + interaction.user.tag })
        .setTimestamp();

      await interaction.reply({ embeds: [serverEmbed] });
    } else if (subcommand === "utilisateur") {
      const targetUser = interaction.options.getUser("cible");
      const member = await interaction.guild.members
        .fetch(targetUser.id)
        .catch(() => null);

      const userEmbed = new EmbedBuilder()
        .setColor(member ? member.displayHexColor : 0x0099ff)
        .setTitle(`👤 Informations sur ${targetUser.username}`)
        .setThumbnail(targetUser.displayAvatarURL({ dynamic: true }))
        .addFields(
          { name: "ID", value: targetUser.id, inline: true },
          {
            name: "Compte créé le",
            value: `<t:${Math.floor(targetUser.createdTimestamp / 1000)}:R>`,
            inline: true,
          }
        );

      // Ajouter des informations spécifiques au membre s'il est sur le serveur
      if (member) {
        userEmbed.addFields(
          {
            name: "A rejoint le serveur",
            value: member.joinedAt
              ? `<t:${Math.floor(member.joinedTimestamp / 1000)}:R>`
              : "Inconnu",
            inline: true,
          },
          { name: "Surnom", value: member.nickname || "Aucun", inline: true },
          {
            name: "Rôles",
            value:
              member.roles.cache.size <= 1
                ? "Aucun"
                : member.roles.cache
                    .filter((r) => r.id !== interaction.guild.id)
                    .map((r) => `<@&${r.id}>`)
                    .join(", "),
          }
        );
      } else {
        userEmbed.setDescription(
          "*Cet utilisateur n'est pas membre de ce serveur.*"
        );
      }

      userEmbed
        .setFooter({ text: "Demandé par " + interaction.user.tag })
        .setTimestamp();

      await interaction.reply({ embeds: [userEmbed] });
    }
  },
};
```

#### Commande Roll (src/commands/roll.js)

```javascript
// src/commands/roll.js
const { SlashCommandBuilder } = require("discord.js");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("roll")
    .setDescription("Lance un dé et donne un résultat aléatoire")
    .addIntegerOption((option) =>
      option
        .setName("faces")
        .setDescription("Nombre de faces du dé (par défaut: 6)")
        .setMinValue(2)
        .setMaxValue(1000)
        .setRequired(false)
    )
    .addIntegerOption((option) =>
      option
        .setName("nombre")
        .setDescription("Nombre de dés à lancer (par défaut: 1)")
        .setMinValue(1)
        .setMaxValue(10)
        .setRequired(false)
    ),

  async execute(interaction) {
    // Récupérer les options ou utiliser les valeurs par défaut
    const faces = interaction.options.getInteger("faces") || 6;
    const nombre = interaction.options.getInteger("nombre") || 1;

    // Générer les résultats
    const resultats = [];
    let total = 0;

    for (let i = 0; i < nombre; i++) {
      const resultat = Math.floor(Math.random() * faces) + 1;
      resultats.push(resultat);
      total += resultat;
    }

    // Construire le message de réponse
    let reponse = `🎲 ${interaction.user} lance ${nombre} dé${
      nombre > 1 ? "s" : ""
    } à ${faces} faces\n\n`;

    if (nombre === 1) {
      reponse += `Résultat : **${resultats[0]}**`;
    } else {
      reponse += `Résultats : ${resultats
        .map((r) => `**${r}**`)
        .join(", ")}\nTotal : **${total}**`;
    }

    // Ajouter une touche d'humour selon le résultat
    if (nombre === 1) {
      if (resultats[0] === 1) {
        reponse += `\n\n😱 Échec critique ! La malchance est avec toi aujourd'hui...`;
      } else if (resultats[0] === faces) {
        reponse += `\n\n🎉 Réussite critique ! Les dieux des dés sont de ton côté !`;
      }
    }

    await interaction.reply(reponse);
  },
};
```

### 4. Utilitaires (src/utils/)

#### Enregistrement des commandes (src/utils/deployCommands.js)

Ce script permet d'enregistrer les commandes de votre bot sur Discord.

```javascript
// src/utils/deployCommands.js
require("dotenv").config();
const { REST, Routes } = require("discord.js");
const fs = require("fs");
const path = require("path");

// Créer un tableau pour stocker les données des commandes
const commands = [];
const commandsPath = path.join(__dirname, "..", "commands");
const commandFiles = fs
  .readdirSync(commandsPath)
  .filter((file) => file.endsWith(".js"));

// Récupérer toutes les commandes
for (const file of commandFiles) {
  const filePath = path.join(commandsPath, file);
  const command = require(filePath);

  if ("data" in command) {
    commands.push(command.data.toJSON());
    console.log(`✓ Commande ajoutée pour déploiement: ${command.data.name}`);
  } else {
    console.warn(`⚠ La commande ${file} n'a pas de propriété 'data'`);
  }
}

// Créer une instance de REST
const rest = new REST({ version: "10" }).setToken(process.env.TOKEN);

// Fonction d'enregistrement
(async () => {
  try {
    console.log(`Démarrage du déploiement de ${commands.length} commandes...`);

    // Enregistrement des commandes (global ou pour un serveur spécifique)
    let data;

    if (process.env.GUILD_ID) {
      // Déploiement sur un serveur spécifique (plus rapide, idéal pour les tests)
      data = await rest.put(
        Routes.applicationGuildCommands(
          process.env.CLIENT_ID,
          process.env.GUILD_ID
        ),
        { body: commands }
      );
      console.log(
        `✅ Commandes déployées sur le serveur ${process.env.GUILD_ID}`
      );
    } else {
      // Déploiement global (peut prendre jusqu'à une heure pour se propager)
      data = await rest.put(Routes.applicationCommands(process.env.CLIENT_ID), {
        body: commands,
      });
      console.log("✅ Commandes déployées globalement");
    }

    console.log(`${data.length} commandes déployées avec succès !`);
  } catch (error) {
    console.error("Erreur lors du déploiement des commandes:", error);
  }
})();
```

### 5. Configuration complète

Mettez à jour votre fichier `.env` avec toutes les variables nécessaires :

```
TOKEN=votre_token_discord
CLIENT_ID=id_de_votre_application
GUILD_ID=id_de_votre_serveur_pour_tests  # Optionnel, pour déployer les commandes sur un serveur spécifique
```

### 6. Scripts dans package.json

Ajoutez ces scripts dans votre fichier `package.json` :

```json
{
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "deploy": "node src/utils/deployCommands.js"
  }
}
```

Si vous utilisez nodemon pour le développement, installez-le :

```bash
npm install --save-dev nodemon
```

## Lancement et déploiement du bot

Pour déployer vos commandes (slash commands) :

```bash
npm run deploy
```

Pour démarrer votre bot :

```bash
npm start
```

Pour le développement avec redémarrage automatique :

```bash
npm run dev
```

## Fonctionnalités avancées

Maintenant que vous avez un bot fonctionnel, ajoutons quelques fonctionnalités plus avancées :

### 1. Système de niveaux et XP

Créons un système simple de niveaux et d'XP pour les utilisateurs qui envoient des messages :

```javascript
// src/utils/levelSystem.js
const fs = require("fs").promises;
const path = require("path");

// Chemin vers le fichier de données
const DATA_PATH = path.join(__dirname, "..", "..", "data", "users.json");

// Fonction pour charger les données utilisateurs
async function loadUserData() {
  try {
    const data = await fs.readFile(DATA_PATH, "utf8");
    return JSON.parse(data);
  } catch (error) {
    // Si le fichier n'existe pas ou est invalide, retourner un objet vide
    return {};
  }
}

// Fonction pour sauvegarder les données utilisateurs
async function saveUserData(userData) {
  // Créer le dossier data s'il n'existe pas
  try {
    await fs.mkdir(path.dirname(DATA_PATH), { recursive: true });
  } catch (error) {
    // Ignorer si le dossier existe déjà
  }

  // Écrire les données
  await fs.writeFile(DATA_PATH, JSON.stringify(userData, null, 2), "utf8");
}

// Calculer le niveau en fonction des points d'XP
function calculateLevel(xp) {
  return Math.floor(0.1 * Math.sqrt(xp));
}

// Calculer l'XP nécessaire pour le prochain niveau
function calculateXpForNextLevel(level) {
  return Math.pow((level + 1) / 0.1, 2);
}

// Ajouter de l'XP à un utilisateur
async function addXp(userId, username, amount) {
  // Charger les données existantes
  const userData = await loadUserData();

  // Initialiser l'utilisateur s'il n'existe pas
  if (!userData[userId]) {
    userData[userId] = {
      id: userId,
      username: username,
      xp: 0,
      level: 0,
      lastMessageTimestamp: 0,
    };
  }

  // Mettre à jour le nom d'utilisateur (il peut changer)
  userData[userId].username = username;

  // Vérifier si l'utilisateur peut gagner de l'XP (anti-spam)
  const now = Date.now();
  const cooldown = 60000; // 1 minute de cooldown

  if (now - userData[userId].lastMessageTimestamp < cooldown) {
    return {
      leveledUp: false,
      newLevel: userData[userId].level,
      xpGained: 0,
    };
  }

  // Mettre à jour l'horodatage du dernier message
  userData[userId].lastMessageTimestamp = now;

  // Ajouter l'XP
  userData[userId].xp += amount;

  // Calculer le nouveau niveau
  const newLevel = calculateLevel(userData[userId].xp);
  const leveledUp = newLevel > userData[userId].level;

  // Mettre à jour le niveau
  userData[userId].level = newLevel;

  // Sauvegarder les données
  await saveUserData(userData);

  return {
    leveledUp,
    newLevel,
    xpGained: amount,
  };
}

// Obtenir les données d'un utilisateur
async function getUserData(userId) {
  const userData = await loadUserData();
  return userData[userId] || null;
}

// Obtenir le classement des utilisateurs
async function getLeaderboard(limit = 10) {
  const userData = await loadUserData();

  return Object.values(userData)
    .sort((a, b) => b.xp - a.xp)
    .slice(0, limit);
}

module.exports = {
  addXp,
  getUserData,
  getLeaderboard,
  calculateLevel,
  calculateXpForNextLevel,
};
```

Maintenant, intégrons ce système dans notre gestionnaire de messages :

```javascript
// src/events/messageCreate.js (mise à jour)
const { EmbedBuilder } = require("discord.js");
const levelSystem = require("../utils/levelSystem");

module.exports = {
  name: "messageCreate",
  once: false,
  async execute(message, client) {
    // Ignorer les messages des bots et les messages dans les DM
    if (message.author.bot || !message.guild) return;

    // Ajouter de l'XP pour chaque message (entre 15 et 25 XP)
    const xpAmount = Math.floor(Math.random() * 11) + 15;
    const result = await levelSystem.addXp(
      message.author.id,
      message.author.username,
      xpAmount
    );

    // Si l'utilisateur a gagné un niveau
    if (result.leveledUp) {
      // Créer un message d'embed pour la montée de niveau
      const levelUpEmbed = new EmbedBuilder()
        .setColor(0x00ff00)
        .setTitle("🎉 Niveau supérieur !")
        .setDescription(
          `Félicitations ${message.author} ! Tu es passé au niveau **${result.newLevel}** !`
        )
        .setThumbnail(message.author.displayAvatarURL({ dynamic: true }))
        .setFooter({ text: "Continue comme ça !" })
        .setTimestamp();

      // Envoyer le message dans le canal actuel
      await message.channel.send({ embeds: [levelUpEmbed] });
    }

    // Réagir à certains mots-clés (exemple simple)
    if (message.content.toLowerCase().includes("bonjour")) {
      await message.reply(
        `Bonjour ${message.author.username} ! Comment vas-tu ?`
      );
    } else if (message.content.toLowerCase().includes("bonne nuit")) {
      await message.reply(`Bonne nuit ${message.author.username} ! À demain !`);
    }

    // Le reste de votre code de gestion des messages...
  },
};
```

Ajoutons une commande pour voir son niveau et celui des autres :

```javascript
// src/commands/niveau.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const levelSystem = require("../utils/levelSystem");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("niveau")
    .setDescription("Affiche votre niveau ou celui d'un autre utilisateur")
    .addUserOption((option) =>
      option
        .setName("utilisateur")
        .setDescription("L'utilisateur dont vous voulez voir le niveau")
        .setRequired(false)
    ),

  async execute(interaction) {
    // Déterminer l'utilisateur cible
    const targetUser =
      interaction.options.getUser("utilisateur") || interaction.user;

    // Obtenir les données de l'utilisateur
    const userData = await levelSystem.getUserData(targetUser.id);

    if (!userData) {
      return interaction.reply({
        content: `${
          targetUser.id === interaction.user.id
            ? "Vous n'avez"
            : "Cet utilisateur n'a"
        } pas encore gagné d'XP sur ce serveur.`,
        ephemeral: true,
      });
    }

    // Calculer l'XP pour le niveau suivant
    const nextLevelXp = levelSystem.calculateXpForNextLevel(userData.level);
    const xpProgress = Math.floor((userData.xp / nextLevelXp) * 100);

    // Créer une barre de progression
    const progressBarLength = 20;
    const filledLength = Math.floor((xpProgress / 100) * progressBarLength);
    const progressBar =
      "█".repeat(filledLength) + "░".repeat(progressBarLength - filledLength);

    // Créer l'embed
    const levelEmbed = new EmbedBuilder()
      .setColor(0x3498db)
      .setTitle(`📊 Niveau de ${targetUser.username}`)
      .setThumbnail(targetUser.displayAvatarURL({ dynamic: true }))
      .addFields(
        { name: "Niveau", value: `${userData.level}`, inline: true },
        {
          name: "XP",
          value: `${userData.xp}/${Math.floor(nextLevelXp)}`,
          inline: true,
        },
        { name: "Progression", value: `${progressBar} ${xpProgress}%` }
      )
      .setFooter({ text: `ID: ${targetUser.id}` })
      .setTimestamp();

    await interaction.reply({ embeds: [levelEmbed] });
  },
};
```

Et une commande pour afficher le classement :

```javascript
// src/commands/classement.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const levelSystem = require("../utils/levelSystem");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("classement")
    .setDescription("Affiche le classement des membres par niveau")
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
    const leaderboardData = await levelSystem.getLeaderboard(limit);

    if (leaderboardData.length === 0) {
      return interaction.reply({
        content: "Aucun membre n'a encore gagné d'XP sur ce serveur.",
        ephemeral: true,
      });
    }

    // Créer la description du classement
    let description = "";

    leaderboardData.forEach((user, index) => {
      // Ajouter des médailles pour les 3 premiers
      let medal = "";
      if (index === 0) medal = "🥇 ";
      else if (index === 1) medal = "🥈 ";
      else if (index === 2) medal = "🥉 ";

      description += `${medal}**#${index + 1}** <@${user.id}> - Niveau ${
        user.level
      } (${user.xp} XP)\n`;
    });

    // Créer l'embed
    const leaderboardEmbed = new EmbedBuilder()
      .setColor(0xf1c40f)
      .setTitle(`🏆 Classement des niveaux - Top ${limit}`)
      .setDescription(description)
      .setFooter({ text: "Continuez à discuter pour gagner de l'XP !" })
      .setTimestamp();

    await interaction.reply({ embeds: [leaderboardEmbed] });
  },
};
```

### 2. Ajout d'un système de musique

La lecture de musique est une fonctionnalité populaire des bots Discord. Voici comment implémenter un système de musique simple :

Tout d'abord, installez les dépendances nécessaires :

```bash
npm install @discordjs/voice @discordjs/opus ytdl-core play-dl
```

Note : Vous aurez peut-être besoin de FFmpeg pour que cela fonctionne correctement. Installez-le sur votre système d'exploitation.

Créons un utilitaire pour gérer la musique :

```javascript
// src/utils/musicPlayer.js
const {
  createAudioPlayer,
  createAudioResource,
  joinVoiceChannel,
  NoSubscriberBehavior,
  AudioPlayerStatus,
  VoiceConnectionStatus,
} = require("@discordjs/voice");
const play = require("play-dl");

// Map pour stocker les joueurs de musique pour chaque serveur
const players = new Map();

// Classe représentant un lecteur de musique pour un serveur
class MusicPlayer {
  constructor(guildId) {
    this.guildId = guildId;
    this.queue = [];
    this.currentSong = null;
    this.connection = null;
    this.player = createAudioPlayer({
      behaviors: {
        noSubscriber: NoSubscriberBehavior.Pause,
      },
    });

    // Configurer les gestionnaires d'événements
    this.player.on(AudioPlayerStatus.Idle, () => this.playNext());
    this.player.on("error", (error) => {
      console.error(`Erreur avec le lecteur audio: ${error.message}`);
      this.playNext();
    });
  }

  // Rejoindre un canal vocal
  join(channel) {
    this.connection = joinVoiceChannel({
      channelId: channel.id,
      guildId: channel.guild.id,
      adapterCreator: channel.guild.voiceAdapterCreator,
    });

    this.connection.on(VoiceConnectionStatus.Disconnected, () => {
      this.leave();
    });

    this.connection.subscribe(this.player);
    return this;
  }

  // Quitter le canal vocal
  leave() {
    if (this.connection) {
      this.player.stop();
      this.connection.destroy();
      this.connection = null;
      this.queue = [];
      this.currentSong = null;
    }

    players.delete(this.guildId);
  }

  // Ajouter une chanson à la file d'attente
  async addToQueue(songInfo, requestedBy) {
    const song = {
      title: songInfo.title,
      url: songInfo.url,
      duration: songInfo.duration,
      thumbnail: songInfo.thumbnail,
      requestedBy: requestedBy,
    };

    this.queue.push(song);

    // Si rien n'est en cours de lecture, démarrer la lecture
    if (!this.currentSong) {
      return this.playNext();
    }

    return song;
  }

  // Passer à la chanson suivante
  async playNext() {
    // Si la file d'attente est vide, arrêter la lecture
    if (this.queue.length === 0) {
      this.currentSong = null;
      return null;
    }

    // Obtenir la prochaine chanson
    const nextSong = this.queue.shift();
    this.currentSong = nextSong;

    try {
      // Obtenir le stream audio
      const stream = await play.stream(nextSong.url);
      const resource = createAudioResource(stream.stream, {
        inputType: stream.type,
        inlineVolume: true,
      });

      // Définir le volume
      resource.volume.setVolume(0.5);

      // Jouer la chanson
      this.player.play(resource);

      return nextSong;
    } catch (error) {
      console.error(`Erreur lors de la lecture de ${nextSong.url}:`, error);
      // Passer à la chanson suivante en cas d'erreur
      return this.playNext();
    }
  }

  // Mettre en pause la lecture
  pause() {
    if (this.player.state.status === AudioPlayerStatus.Playing) {
      this.player.pause();
      return true;
    }
    return false;
  }

  // Reprendre la lecture
  resume() {
    if (this.player.state.status === AudioPlayerStatus.Paused) {
      this.player.unpause();
      return true;
    }
    return false;
  }

  // Arrêter la lecture et vider la file d'attente
  stop() {
    this.queue = [];
    this.player.stop();
    this.currentSong = null;
    return true;
  }

  // Passer la chanson actuelle
  skip() {
    if (this.currentSong) {
      this.player.stop();
      return true;
    }
    return false;
  }

  // Obtenir la file d'attente
  getQueue() {
    return {
      current: this.currentSong,
      upcoming: this.queue,
    };
  }

  // Vérifier l'état du lecteur
  getStatus() {
    return this.player.state.status;
  }
}

// Fonction pour obtenir ou créer un lecteur de musique pour un serveur
function getPlayer(guildId) {
  if (!players.has(guildId)) {
    players.set(guildId, new MusicPlayer(guildId));
  }
  return players.get(guildId);
}

module.exports = {
  getPlayer,
};
```

Maintenant, ajoutons les commandes pour contrôler la lecture de musique :

```javascript
// src/commands/play.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");
const play = require("play-dl");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("play")
    .setDescription("Joue une musique depuis YouTube")
    .addStringOption((option) =>
      option
        .setName("recherche")
        .setDescription("Lien YouTube ou termes de recherche")
        .setRequired(true)
    ),

  async execute(interaction) {
    // Vérifier si l'utilisateur est dans un canal vocal
    if (!interaction.member.voice.channel) {
      return interaction.reply({
        content:
          "Vous devez être dans un canal vocal pour utiliser cette commande !",
        ephemeral: true,
      });
    }

    // Obtenir le terme de recherche
    const searchTerm = interaction.options.getString("recherche");

    await interaction.deferReply();

    try {
      // Obtenir les informations de la vidéo
      let songInfo;

      // Vérifier si c'est un lien YouTube
      if (play.yt_validate(searchTerm) === "video") {
        const videoInfo = await play.video_info(searchTerm);
        songInfo = {
          title: videoInfo.video_details.title,
          url: videoInfo.video_details.url,
          duration: videoInfo.video_details.durationInSec,
          thumbnail: videoInfo.video_details.thumbnails[0].url,
        };
      } else {
        // C'est un terme de recherche
        const searchResults = await play.search(searchTerm, { limit: 1 });

        if (searchResults.length === 0) {
          return interaction.editReply(
            "❌ Aucun résultat trouvé pour cette recherche."
          );
        }

        songInfo = {
          title: searchResults[0].title,
          url: searchResults[0].url,
          duration: searchResults[0].durationInSec,
          thumbnail: searchResults[0].thumbnails[0].url,
        };
      }

      // Formater la durée
      const minutes = Math.floor(songInfo.duration / 60);
      const seconds = songInfo.duration % 60;
      const formattedDuration = `${minutes}:${
        seconds < 10 ? "0" : ""
      }${seconds}`;

      // Obtenir le lecteur de musique pour ce serveur
      const player = musicPlayer.getPlayer(interaction.guildId);

      // Rejoindre le canal vocal si ce n'est pas déjà fait
      if (!player.connection) {
        player.join(interaction.member.voice.channel);
      }

      // Ajouter la chanson à la file d'attente
      const song = await player.addToQueue(songInfo, interaction.user.tag);

      // Créer un embed pour afficher les informations
      const embed = new EmbedBuilder()
        .setColor(0x3498db)
        .setTitle(song.title)
        .setURL(song.url)
        .setDescription(`Durée: ${formattedDuration}`)
        .setThumbnail(song.thumbnail)
        .setFooter({ text: `Demandé par ${interaction.user.tag}` });

      if (player.queue.length === 0) {
        embed.setAuthor({ name: "En lecture maintenant" });
      } else {
        embed.setAuthor({
          name: `Ajouté à la file d'attente (Position: ${player.queue.length})`,
        });
      }

      await interaction.editReply({ embeds: [embed] });
    } catch (error) {
      console.error("Erreur lors de la lecture de musique:", error);
      await interaction.editReply(
        "❌ Une erreur s'est produite lors du traitement de votre demande."
      );
    }
  },
};
```

Ajoutons d'autres commandes pour contrôler la musique :

```javascript
// src/commands/queue.js
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("queue")
    .setDescription("Affiche la file d'attente de musique"),

  async execute(interaction) {
    // Obtenir le lecteur de musique pour ce serveur
    const player = musicPlayer.getPlayer(interaction.guildId);
    const queueInfo = player.getQueue();

    if (!queueInfo.current) {
      return interaction.reply({
        content: "🔇 Aucune musique n'est en cours de lecture.",
        ephemeral: true,
      });
    }

    // Formater la durée
    function formatDuration(seconds) {
      const minutes = Math.floor(seconds / 60);
      const remainingSeconds = seconds % 60;
      return `${minutes}:${
        remainingSeconds < 10 ? "0" : ""
      }${remainingSeconds}`;
    }

    // Créer l'embed pour la file d'attente
    const embed = new EmbedBuilder()
      .setColor(0x3498db)
      .setTitle("🎵 File d'attente de musique")
      .setThumbnail(queueInfo.current.thumbnail);

    // Ajouter la chanson en cours
    embed.addFields({
      name: "🎧 En cours de lecture",
      value: `[${queueInfo.current.title}](${
        queueInfo.current.url
      }) - ${formatDuration(queueInfo.current.duration)}\nDemandé par ${
        queueInfo.current.requestedBy
      }`,
    });

    // Ajouter les prochaines chansons s'il y en a
    if (queueInfo.upcoming.length > 0) {
      const upcomingSongs = queueInfo.upcoming
        .slice(0, 10)
        .map(
          (song, index) =>
            `${index + 1}. [${song.title}](${song.url}) - ${formatDuration(
              song.duration
            )}`
        )
        .join("\n");

      embed.addFields({
        name: `📋 Prochaines chansons (${queueInfo.upcoming.length})`,
        value: upcomingSongs,
      });

      if (queueInfo.upcoming.length > 10) {
        embed.setFooter({
          text: `Et ${queueInfo.upcoming.length - 10} autres chansons...`,
        });
      }
    } else {
      embed.addFields({
        name: "📋 Prochaines chansons",
        value: "Aucune chanson dans la file d'attente",
      });
    }

    await interaction.reply({ embeds: [embed] });
  },
};
```

```javascript
// src/commands/skip.js
const { SlashCommandBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("skip")
    .setDescription("Passe à la chanson suivante"),

  async execute(interaction) {
    // Vérifier si l'utilisateur est dans un canal vocal
    if (!interaction.member.voice.channel) {
      return interaction.reply({
        content:
          "Vous devez être dans un canal vocal pour utiliser cette commande !",
        ephemeral: true,
      });
    }

    // Obtenir le lecteur de musique pour ce serveur
    const player = musicPlayer.getPlayer(interaction.guildId);

    // Vérifier s'il y a une chanson en cours
    if (!player.currentSong) {
      return interaction.reply({
        content: "🔇 Aucune musique n'est en cours de lecture.",
        ephemeral: true,
      });
    }

    // Stocker le titre de la chanson actuelle
    const currentTitle = player.currentSong.title;

    // Passer à la chanson suivante
    const skipped = player.skip();

    if (skipped) {
      await interaction.reply(
        `⏭️ **${interaction.user.username}** a passé la chanson: **${currentTitle}**`
      );
    } else {
      await interaction.reply({
        content: "❌ Impossible de passer la chanson.",
        ephemeral: true,
      });
    }
  },
};
```

```javascript
// src/commands/stop.js
const { SlashCommandBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("stop")
    .setDescription("Arrête la musique et vide la file d'attente"),

  async execute(interaction) {
    // Vérifier si l'utilisateur est dans un canal vocal
    if (!interaction.member.voice.channel) {
      return interaction.reply({
        content:
          "Vous devez être dans un canal vocal pour utiliser cette commande !",
        ephemeral: true,
      });
    }

    // Obtenir le lecteur de musique pour ce serveur
    const player = musicPlayer.getPlayer(interaction.guildId);

    // Vérifier s'il y a une chanson en cours
    if (!player.currentSong) {
      return interaction.reply({
        content: "🔇 Aucune musique n'est en cours de lecture.",
        ephemeral: true,
      });
    }

    // Arrêter la musique
    player.stop();

    await interaction.reply(
      `⏹️ **${interaction.user.username}** a arrêté la musique et vidé la file d'attente.`
    );
  },
};
```

```javascript
// src/commands/pause.js
const { SlashCommandBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");
const { AudioPlayerStatus } = require("@discordjs/voice");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("pause")
    .setDescription("Met en pause ou reprend la lecture de la musique"),

  async execute(interaction) {
    // Vérifier si l'utilisateur est dans un canal vocal
    if (!interaction.member.voice.channel) {
      return interaction.reply({
        content:
          "Vous devez être dans un canal vocal pour utiliser cette commande !",
        ephemeral: true,
      });
    }

    // Obtenir le lecteur de musique pour ce serveur
    const player = musicPlayer.getPlayer(interaction.guildId);

    // Vérifier s'il y a une chanson en cours
    if (!player.currentSong) {
      return interaction.reply({
        content: "🔇 Aucune musique n'est en cours de lecture.",
        ephemeral: true,
      });
    }

    // Basculer entre pause et reprise
    if (player.getStatus() === AudioPlayerStatus.Playing) {
      player.pause();
      await interaction.reply(
        `⏸️ **${interaction.user.username}** a mis la musique en pause.`
      );
    } else if (player.getStatus() === AudioPlayerStatus.Paused) {
      player.resume();
      await interaction.reply(
        `▶️ **${interaction.user.username}** a repris la lecture de la musique.`
      );
    } else {
      await interaction.reply({
        content: "❌ Impossible de basculer l'état de la lecture.",
        ephemeral: true,
      });
    }
  },
};
```

```javascript
// src/commands/leave.js
const { SlashCommandBuilder } = require("discord.js");
const musicPlayer = require("../utils/musicPlayer");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("leave")
    .setDescription("Fait quitter le bot du canal vocal"),

  async execute(interaction) {
    // Vérifier si l'utilisateur est dans un canal vocal
    if (!interaction.member.voice.channel) {
      return interaction.reply({
        content:
          "Vous devez être dans un canal vocal pour utiliser cette commande !",
        ephemeral: true,
      });
    }

    // Obtenir le lecteur de musique pour ce serveur
    const player = musicPlayer.getPlayer(interaction.guildId);

    // Vérifier si le bot est dans un canal vocal
    if (!player.connection) {
      return interaction.reply({
        content: "Je ne suis pas dans un canal vocal.",
        ephemeral: true,
      });
    }

    // Faire quitter le bot
    player.leave();

    await interaction.reply(
      `👋 **${interaction.user.username}** m'a fait quitter le canal vocal.`
    );
  },
};
```

## Bonnes pratiques pour le développement de bots Discord

1. **Sécurité**

   - Ne jamais intégrer directement votre token dans le code
   - Utiliser les permissions minimales nécessaires pour votre bot
   - Faire attention aux injections de commande

2. **Performance**

   - Optimiser les opérations coûteuses (requêtes API, opérations de base de données)
   - Limiter le nombre de messages envoyés en rafale
   - Utiliser le caching quand c'est approprié

3. **Maintenabilité**

   - Organiser votre code en modules réutilisables
   - Documenter votre code avec des commentaires
   - Utiliser la gestion d'erreurs appropriée
   - Implémenter un système de logging

4. **Expérience utilisateur**

   - Utiliser les embeds pour des messages plus attrayants
   - Fournir des retours clairs aux utilisateurs
   - Inclure des commandes d'aide détaillées
   - Utiliser des réactions pour l'interactivité

5. **Déploiement**
   - Utiliser un service d'hébergement fiable (Heroku, Railway, DigitalOcean)
   - Mettre en place un système de monitoring
   - Sauvegarder régulièrement vos données

## Déploiement de votre bot

Pour maintenir votre bot en ligne 24/7, vous devrez le déployer sur un serveur. Voici quelques options populaires :

### Heroku

1. Créez un compte sur [Heroku](https://heroku.com)
2. Installez Heroku CLI
3. Créez un fichier `Procfile` à la racine de votre projet :
   ```
   worker: node src/index.js
   ```
4. Initialisez un dépôt Git et poussez vers Heroku :
   ```bash
   git init
   heroku create
   git add .
   git commit -m "Initial commit"
   git push heroku main
   ```
5. Configurez les variables d'environnement sur Heroku :
   ```bash
   heroku config:set TOKEN=votre_token_discord
   heroku config:set CLIENT_ID=id_de_votre_application
   ```
6. Lancez le worker :
   ```bash
   heroku ps:scale worker=1
   ```

### Railway

1. Créez un compte sur [Railway](https://railway.app)
2. Connectez votre dépôt GitHub
3. Créez un nouveau projet depuis votre dépôt
4. Configurez les variables d'environnement dans l'interface
5. Railway détectera automatiquement la commande de démarrage de votre `package.json`

### VPS (Digital Ocean, AWS, etc.)

Si vous préférez un contrôle total, vous pouvez déployer sur un VPS :

1. Connectez-vous à votre serveur via SSH
2. Installez Node.js et npm
3. Clonez votre dépôt
4. Installez PM2 pour gérer votre application :
   ```bash
   npm install -g pm2
   pm2 start src/index.js --name "mon-bot-discord"
   pm2 save
   pm2 startup
   ```

## Conclusion

Dans ce chapitre, nous avons vu comment :

- Créer une application Discord et obtenir un token
- Configurer un projet Node.js pour un bot Discord
- Implémenter un système de commandes slash
- Gérer différents événements Discord
- Créer des fonctionnalités avancées comme un système de niveaux et un lecteur de musique
- Déployer votre bot pour qu'il reste en ligne

Votre bot Discord est maintenant prêt à être utilisé et personnalisé selon vos besoins spécifiques. Dans le prochain chapitre, nous explorerons comment créer un bot Twitch en utilisant les connaissances que vous avez acquises jusqu'à présent.

## Ressources supplémentaires

- [Documentation officielle Discord.js](https://discord.js.org/)
- [Guide Discord.js](https://discordjs.guide/)
- [API Discord](https://discord.com/developers/docs/intro)
- [Discord.js GitHub Repository](https://github.com/discordjs/discord.js)
- [Communauté Discord.js](https://discord.gg/djs)
