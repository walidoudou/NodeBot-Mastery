# Chapitre 7 - Fonctionnalités avancées pour les bots

Dans ce chapitre, nous allons explorer des fonctionnalités avancées pour rendre vos bots Discord et Twitch plus puissants, interactifs et utiles. Ces techniques vous permettront de créer des expériences plus engageantes pour les utilisateurs et d'automatiser des tâches complexes.

## Systèmes de modération automatique

La modération est l'une des fonctionnalités les plus importantes pour les bots de communauté. Voyons comment implémenter un système de modération efficace.

### Détection et filtrage de contenu indésirable

```javascript
// src/utils/autoMod.js
const { EmbedBuilder } = require("discord.js");

class AutoModerator {
  constructor() {
    // Liste de mots interdits (à personnaliser)
    this.badWords = [
      "badword1",
      "badword2",
      "badword3",
      // Ajoutez vos mots interdits ici
    ];

    // Regex pour détecter les invitations Discord
    this.inviteRegex =
      /(discord\.(gg|io|me|li)|discordapp\.com\/invite)\/[a-zA-Z0-9]+/i;

    // Regex pour détecter le spam de majuscules
    this.capsRegex = /[A-Z]{10,}/;

    // Regex pour détecter le spam d'emojis
    this.emojiSpamRegex = /(<a?:.+?:\d+>|\p{Emoji}){5,}/u;
  }

  // Vérifier si un message contient du contenu inapproprié
  checkMessage(message, config = {}) {
    const content = message.content.toLowerCase();
    const violations = [];

    // Vérifier les mots interdits
    if (config.filterBadWords !== false) {
      for (const word of this.badWords) {
        const regex = new RegExp(`\\b${word}\\b`, "i");
        if (regex.test(content)) {
          violations.push({
            type: "BAD_WORD",
            word: word,
            severity: "MEDIUM",
          });
        }
      }
    }

    // Vérifier les invitations Discord
    if (config.filterInvites !== false) {
      if (this.inviteRegex.test(content)) {
        violations.push({
          type: "INVITE_LINK",
          severity: "HIGH",
        });
      }
    }

    // Vérifier le spam de majuscules
    if (config.filterCaps !== false) {
      if (this.capsRegex.test(message.content)) {
        violations.push({
          type: "EXCESSIVE_CAPS",
          severity: "LOW",
        });
      }
    }

    // Vérifier le spam d'emojis
    if (config.filterEmojiSpam !== false) {
      if (this.emojiSpamRegex.test(message.content)) {
        violations.push({
          type: "EMOJI_SPAM",
          severity: "LOW",
        });
      }
    }

    // Vérifier les messages identiques répétés (nécessite un suivi des messages précédents)
    if (config.filterSpam !== false && this.isSpam(message)) {
      violations.push({
        type: "MESSAGE_SPAM",
        severity: "MEDIUM",
      });
    }

    return {
      hasViolations: violations.length > 0,
      violations: violations,
    };
  }

  // Détection de spam (implémentation simple)
  isSpam(message) {
    // Cette méthode nécessiterait une implémentation plus complexe
    // avec suivi des messages précédents par utilisateur
    return false;
  }

  // Appliquer des actions de modération automatiques
  async applyAutoModAction(message, violations, config = {}) {
    // Actions possibles: NONE, DELETE, WARN, TIMEOUT, KICK, BAN
    const highestSeverity = violations.reduce((highest, v) => {
      const severityMap = { LOW: 1, MEDIUM: 2, HIGH: 3 };
      return Math.max(highest, severityMap[v.severity] || 0);
    }, 0);

    // Déterminer l'action en fonction de la sévérité
    let action = "NONE";

    if (highestSeverity === 1) action = config.lowSeverityAction || "DELETE";
    else if (highestSeverity === 2)
      action = config.mediumSeverityAction || "DELETE";
    else if (highestSeverity === 3)
      action = config.highSeverityAction || "DELETE";

    // Appliquer l'action
    switch (action) {
      case "DELETE":
        // Supprimer le message
        try {
          await message.delete();
          console.log(
            `AutoMod: Message de ${
              message.author.tag
            } supprimé pour violation(s): ${violations
              .map((v) => v.type)
              .join(", ")}`
          );
        } catch (error) {
          console.error("Erreur lors de la suppression du message:", error);
        }
        break;

      case "WARN":
        // Avertir l'utilisateur
        try {
          await message.delete();
          const warnEmbed = new EmbedBuilder()
            .setColor("#FFA500")
            .setTitle("⚠️ Avertissement de modération")
            .setDescription(
              `Votre message a été supprimé car il enfreint nos règles.`
            )
            .addFields({
              name: "Raison(s)",
              value: violations
                .map((v) => {
                  switch (v.type) {
                    case "BAD_WORD":
                      return "Langage inapproprié";
                    case "INVITE_LINK":
                      return "Lien d'invitation non autorisé";
                    case "EXCESSIVE_CAPS":
                      return "Utilisation excessive de majuscules";
                    case "EMOJI_SPAM":
                      return "Spam d'emojis";
                    case "MESSAGE_SPAM":
                      return "Spam de messages";
                    default:
                      return "Violation des règles";
                  }
                })
                .join(", "),
            })
            .setFooter({
              text: "La modération automatique est active sur ce serveur",
            });

          await message.author.send({ embeds: [warnEmbed] }).catch(() => {
            // L'utilisateur a peut-être désactivé les DM
            console.log(
              `Impossible d'envoyer un avertissement à ${message.author.tag}`
            );
          });
        } catch (error) {
          console.error(
            "Erreur lors de l'avertissement de l'utilisateur:",
            error
          );
        }
        break;

      case "TIMEOUT":
        // Discord: Mettre l'utilisateur en timeout
        if (message.guild && message.member) {
          try {
            await message.delete();
            await message.member.timeout(
              5 * 60 * 1000,
              "Violation des règles de modération automatique"
            );
            console.log(
              `AutoMod: Timeout appliqué à ${
                message.author.tag
              } pour violation(s): ${violations.map((v) => v.type).join(", ")}`
            );
          } catch (error) {
            console.error("Erreur lors de l'application du timeout:", error);
          }
        }
        break;

      // Autres actions comme KICK et BAN pourraient être implémentées de manière similaire
    }

    // Journaliser la violation
    this.logViolation(message, violations, action);

    return action;
  }

  // Journaliser les violations
  logViolation(message, violations, action) {
    // Dans une implémentation complète, vous pourriez enregistrer cela dans une base de données
    console.log(
      `[AutoMod] ${new Date().toISOString()} - User: ${
        message.author.tag
      }, Action: ${action}, Violations: ${JSON.stringify(violations)}`
    );

    // Vous pourriez également envoyer un log dans un canal spécifique
    if (
      message.guild &&
      message.guild.channels.cache.has(config.logChannelId)
    ) {
      const logChannel = message.guild.channels.cache.get(config.logChannelId);

      const logEmbed = new EmbedBuilder()
        .setColor("#FF0000")
        .setTitle("📝 Log de modération automatique")
        .setDescription(`Un message de ${message.author} a été modéré.`)
        .addFields(
          {
            name: "Utilisateur",
            value: `${message.author.tag} (${message.author.id})`,
          },
          { name: "Canal", value: `<#${message.channel.id}>` },
          { name: "Action", value: action },
          {
            name: "Violations",
            value: violations
              .map((v) => `${v.type} (${v.severity})`)
              .join("\n"),
          },
          {
            name: "Message",
            value:
              message.content.length > 1024
                ? message.content.substring(0, 1021) + "..."
                : message.content,
          }
        )
        .setTimestamp();

      logChannel.send({ embeds: [logEmbed] }).catch(console.error);
    }
  }
}

module.exports = new AutoModerator();
```

### Intégration du système de modération automatique dans le bot Discord

```javascript
// Ajoutez dans src/events/messageCreate.js (Discord)
const autoMod = require("../utils/autoMod");
const { PermissionFlagsBits } = require("discord.js");

module.exports = {
  name: "messageCreate",
  once: false,
  async execute(message, client) {
    // Ignorer les messages des bots
    if (message.author.bot) return;

    // Ignorer les messages des modérateurs/administrateurs
    if (
      message.member &&
      (message.member.permissions.has(PermissionFlagsBits.ManageMessages) ||
        message.member.permissions.has(PermissionFlagsBits.Administrator))
    ) {
      // Les modérateurs sont exemptés de la modération automatique
      // Continuer avec le traitement normal du message
    } else {
      // Vérifier le message avec le système de modération automatique
      const modResult = autoMod.checkMessage(message, {
        filterBadWords: true,
        filterInvites: true,
        filterCaps: true,
        filterEmojiSpam: true,
        filterSpam: true,
        lowSeverityAction: "DELETE",
        mediumSeverityAction: "WARN",
        highSeverityAction: "TIMEOUT",
      });

      // Si des violations sont détectées, appliquer l'action de modération
      if (modResult.hasViolations) {
        const action = await autoMod.applyAutoModAction(
          message,
          modResult.violations
        );

        // Si le message a été supprimé, arrêter le traitement
        if (action !== "NONE") {
          return;
        }
      }
    }

    // Continuer avec le traitement normal du message
    // (vérifier les commandes, etc.)
  },
};
```

### Système d'avertissements et sanctions progressives

```javascript
// src/utils/warningSystem.js
const { EmbedBuilder } = require("discord.js");

class WarningSystem {
  constructor(database) {
    this.db = database;

    // Créer la table des avertissements si elle n'existe pas
    this.db.exec(`
            CREATE TABLE IF NOT EXISTS warnings (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT NOT NULL,
                guild_id TEXT NOT NULL,
                moderator_id TEXT NOT NULL,
                reason TEXT,
                timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                active BOOLEAN DEFAULT TRUE
            )
        `);
  }

  // Ajouter un avertissement
  async addWarning(userId, guildId, moderatorId, reason) {
    const stmt = this.db.prepare(`
            INSERT INTO warnings (user_id, guild_id, moderator_id, reason)
            VALUES (?, ?, ?, ?)
        `);

    const result = stmt.run(userId, guildId, moderatorId, reason);

    // Vérifier s'il faut appliquer des sanctions automatiques
    await this.checkForAutomaticSanctions(userId, guildId);

    return {
      id: result.lastInsertRowid,
      userId,
      guildId,
      moderatorId,
      reason,
      timestamp: new Date(),
    };
  }

  // Obtenir les avertissements d'un utilisateur
  getWarnings(userId, guildId, activeOnly = true) {
    let sql = `
            SELECT * FROM warnings 
            WHERE user_id = ? AND guild_id = ?
        `;

    if (activeOnly) {
      sql += " AND active = TRUE";
    }

    sql += " ORDER BY timestamp DESC";

    const stmt = this.db.prepare(sql);
    return stmt.all(userId, guildId);
  }

  // Supprimer/pardonner un avertissement
  removeWarning(warningId) {
    const stmt = this.db.prepare(`
            UPDATE warnings
            SET active = FALSE
            WHERE id = ?
        `);

    const result = stmt.run(warningId);
    return result.changes > 0;
  }

  // Vérifier s'il faut appliquer des sanctions automatiques
  async checkForAutomaticSanctions(userId, guildId) {
    const activeWarnings = this.getWarnings(userId, guildId, true);
    const warningCount = activeWarnings.length;

    // Définir les seuils pour les sanctions
    // Ceci est un exemple, à adapter selon vos besoins
    if (warningCount === 3) {
      // Appliquer un timeout de 10 minutes
      return {
        action: "TIMEOUT",
        duration: 10 * 60 * 1000, // 10 minutes en ms
        reason: "Accumulation de 3 avertissements",
      };
    } else if (warningCount === 5) {
      // Expulsion (kick)
      return {
        action: "KICK",
        reason: "Accumulation de 5 avertissements",
      };
    } else if (warningCount >= 7) {
      // Bannissement
      return {
        action: "BAN",
        reason: "Accumulation de 7 avertissements ou plus",
      };
    }

    return { action: "NONE" };
  }

  // Créer un embed pour afficher les avertissements
  createWarningsEmbed(userId, username, warnings) {
    const embed = new EmbedBuilder()
      .setColor("#FF9900")
      .setTitle(`Avertissements de ${username}`)
      .setDescription(`${warnings.length} avertissement(s) actif(s)`)
      .setTimestamp();

    if (warnings.length === 0) {
      embed.addFields({
        name: "Information",
        value: "Cet utilisateur n'a aucun avertissement actif.",
      });
    } else {
      warnings.forEach((warning, index) => {
        const date = new Date(warning.timestamp).toLocaleString();
        embed.addFields({
          name: `Avertissement #${index + 1} (ID: ${warning.id})`,
          value: `**Raison:** ${
            warning.reason || "Non spécifiée"
          }\n**Modérateur:** <@${warning.moderator_id}>\n**Date:** ${date}`,
        });
      });
    }

    return embed;
  }
}

module.exports = WarningSystem;
```

### Commande d'avertissement pour Discord

```javascript
// src/commands/warn.js (Discord)
const { SlashCommandBuilder, PermissionFlagsBits } = require("discord.js");
const WarningSystem = require("../utils/warningSystem");
const database = require("../utils/database");

// Créer une instance du système d'avertissements
const warningSystem = new WarningSystem(database.db);

module.exports = {
  data: new SlashCommandBuilder()
    .setName("warn")
    .setDescription("Ajoute un avertissement à un utilisateur")
    .addUserOption((option) =>
      option
        .setName("utilisateur")
        .setDescription("L'utilisateur à avertir")
        .setRequired(true)
    )
    .addStringOption((option) =>
      option
        .setName("raison")
        .setDescription("La raison de l'avertissement")
        .setRequired(false)
    )
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),

  async execute(interaction) {
    // Obtenir l'utilisateur cible et la raison
    const targetUser = interaction.options.getUser("utilisateur");
    const reason =
      interaction.options.getString("raison") || "Aucune raison spécifiée";

    // Vérifier si l'utilisateur cible est un administrateur
    const targetMember = await interaction.guild.members
      .fetch(targetUser.id)
      .catch(() => null);

    if (!targetMember) {
      return interaction.reply({
        content: "Cet utilisateur n'est pas membre du serveur.",
        ephemeral: true,
      });
    }

    if (targetMember.permissions.has(PermissionFlagsBits.Administrator)) {
      return interaction.reply({
        content: "Vous ne pouvez pas avertir un administrateur.",
        ephemeral: true,
      });
    }

    // Ajouter l'avertissement
    const warning = await warningSystem.addWarning(
      targetUser.id,
      interaction.guild.id,
      interaction.user.id,
      reason
    );

    // Notifier l'utilisateur averti
    try {
      await targetUser
        .send({
          content: `Vous avez reçu un avertissement sur le serveur **${interaction.guild.name}**.\nRaison: ${reason}`,
        })
        .catch(() => {
          // L'utilisateur a peut-être désactivé les DM
          console.log(`Impossible d'envoyer un message à ${targetUser.tag}`);
        });
    } catch (error) {
      console.error("Erreur lors de l'envoi du DM d'avertissement:", error);
    }

    // Vérifier s'il y a des sanctions automatiques à appliquer
    const sanction = await warningSystem.checkForAutomaticSanctions(
      targetUser.id,
      interaction.guild.id
    );

    // Appliquer la sanction si nécessaire
    let sanctionMessage = "";

    if (sanction.action === "TIMEOUT") {
      try {
        await targetMember.timeout(sanction.duration, sanction.reason);
        sanctionMessage = `\nL'utilisateur a également reçu un timeout de ${
          sanction.duration / 60000
        } minutes pour accumulation d'avertissements.`;
      } catch (error) {
        console.error("Erreur lors de l'application du timeout:", error);
      }
    } else if (sanction.action === "KICK") {
      try {
        await targetMember.kick(sanction.reason);
        sanctionMessage = `\nL'utilisateur a été expulsé du serveur pour accumulation d'avertissements.`;
      } catch (error) {
        console.error("Erreur lors de l'expulsion:", error);
      }
    } else if (sanction.action === "BAN") {
      try {
        await interaction.guild.members.ban(targetUser.id, {
          reason: sanction.reason,
        });
        sanctionMessage = `\nL'utilisateur a été banni du serveur pour accumulation d'avertissements.`;
      } catch (error) {
        console.error("Erreur lors du bannissement:", error);
      }
    }

    // Répondre à l'interaction
    await interaction.reply({
      content: `✅ Avertissement ajouté pour ${targetUser}.\nRaison: ${reason}${sanctionMessage}`,
      ephemeral: false, // Visible par tous pour la transparence de modération
    });
  },
};
```

### Commande pour voir les avertissements

```javascript
// src/commands/warnings.js (Discord)
const { SlashCommandBuilder, PermissionFlagsBits } = require("discord.js");
const WarningSystem = require("../utils/warningSystem");
const database = require("../utils/database");

// Créer une instance du système d'avertissements
const warningSystem = new WarningSystem(database.db);

module.exports = {
  data: new SlashCommandBuilder()
    .setName("warnings")
    .setDescription("Affiche les avertissements d'un utilisateur")
    .addUserOption((option) =>
      option
        .setName("utilisateur")
        .setDescription(
          "L'utilisateur dont vous voulez voir les avertissements"
        )
        .setRequired(true)
    )
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),

  async execute(interaction) {
    // Obtenir l'utilisateur cible
    const targetUser = interaction.options.getUser("utilisateur");

    // Récupérer les avertissements
    const warnings = warningSystem.getWarnings(
      targetUser.id,
      interaction.guild.id
    );

    // Créer et envoyer l'embed
    const embed = warningSystem.createWarningsEmbed(
      targetUser.id,
      targetUser.username,
      warnings
    );

    await interaction.reply({
      embeds: [embed],
      ephemeral: true,
    });
  },
};
```

## Intégration des API externes

L'intégration d'API externes peut considérablement enrichir les fonctionnalités de votre bot. Voici quelques exemples :

### Client API générique

```javascript
// src/utils/apiClient.js
const fetch = require("node-fetch");

class ApiClient {
  constructor(baseUrl, options = {}) {
    this.baseUrl = baseUrl;
    this.defaultHeaders = options.headers || {};
    this.timeout = options.timeout || 10000; // 10 secondes par défaut
  }

  // Paramètres génériques pour fetch
  _createFetchOptions(method, headers = {}, body = null) {
    const options = {
      method,
      headers: { ...this.defaultHeaders, ...headers },
      timeout: this.timeout,
    };

    if (body) {
      options.body = typeof body === "string" ? body : JSON.stringify(body);
      if (!options.headers["Content-Type"]) {
        options.headers["Content-Type"] = "application/json";
      }
    }

    return options;
  }

  // Créer l'URL complète
  _createUrl(endpoint, params = {}) {
    const url = new URL(`${this.baseUrl}${endpoint}`);

    // Ajouter les paramètres à l'URL
    Object.keys(params).forEach((key) => {
      url.searchParams.append(key, params[key]);
    });

    return url.toString();
  }

  // Traiter la réponse
  async _handleResponse(response) {
    // Vérifier si la réponse est un succès (code 2xx)
    if (!response.ok) {
      const errorBody = await response.text().catch(() => null);

      throw new Error(
        `API request failed with status ${response.status}: ${
          errorBody || response.statusText
        }`
      );
    }

    // Essayer de parser le corps en JSON, sinon retourner le texte
    try {
      return await response.json();
    } catch (error) {
      return await response.text();
    }
  }

  // Méthode GET
  async get(endpoint, params = {}, headers = {}) {
    const url = this._createUrl(endpoint, params);
    const options = this._createFetchOptions("GET", headers);

    const response = await fetch(url, options);
    return this._handleResponse(response);
  }

  // Méthode POST
  async post(endpoint, body = {}, headers = {}) {
    const url = this._createUrl(endpoint);
    const options = this._createFetchOptions("POST", headers, body);

    const response = await fetch(url, options);
    return this._handleResponse(response);
  }

  // Méthode PUT
  async put(endpoint, body = {}, headers = {}) {
    const url = this._createUrl(endpoint);
    const options = this._createFetchOptions("PUT", headers, body);

    const response = await fetch(url, options);
    return this._handleResponse(response);
  }

  // Méthode DELETE
  async delete(endpoint, headers = {}) {
    const url = this._createUrl(endpoint);
    const options = this._createFetchOptions("DELETE", headers);

    const response = await fetch(url, options);
    return this._handleResponse(response);
  }
}

module.exports = ApiClient;
```

### Exemple d'intégration avec l'API OpenWeatherMap

```javascript
// src/apis/weatherApi.js
const ApiClient = require("../utils/apiClient");

class WeatherApi {
  constructor(apiKey) {
    this.apiClient = new ApiClient("https://api.openweathermap.org/data/2.5/", {
      headers: {
        Accept: "application/json",
      },
    });

    this.apiKey = apiKey;
  }

  // Obtenir la météo actuelle par ville
  async getCurrentWeather(city, units = "metric") {
    return this.apiClient.get("weather", {
      q: city,
      appid: this.apiKey,
      units,
    });
  }

  // Obtenir la météo actuelle par coordonnées
  async getCurrentWeatherByCoords(lat, lon, units = "metric") {
    return this.apiClient.get("weather", {
      lat,
      lon,
      appid: this.apiKey,
      units,
    });
  }

  // Obtenir les prévisions sur 5 jours
  async getForecast(city, units = "metric") {
    return this.apiClient.get("forecast", {
      q: city,
      appid: this.apiKey,
      units,
    });
  }

  // Convertir le code météo en emoji
  getWeatherEmoji(weatherCode) {
    // Correspondances des codes météo d'OpenWeatherMap
    const weatherEmojis = {
      // Groupe 2xx: Orages
      2: "⛈️",
      // Groupe 3xx: Bruine
      3: "🌧️",
      // Groupe 5xx: Pluie
      5: "🌧️",
      // Groupe 6xx: Neige
      6: "❄️",
      // Groupe 7xx: Atmosphère (brouillard, etc.)
      7: "🌫️",
      // Groupe 800: Ciel dégagé
      800: "☀️",
      // Groupe 80x: Nuages
      801: "🌤️", // Quelques nuages
      802: "⛅", // Nuages épars
      803: "🌥️", // Nuages fragmentés
      804: "☁️", // Ciel couvert
    };

    const code = weatherCode.toString();

    // Essayer de trouver un code exact
    if (weatherEmojis[code]) {
      return weatherEmojis[code];
    }

    // Sinon, utiliser le premier chiffre pour un groupe général
    return weatherEmojis[code[0]] || "🌡️";
  }

  // Formater le résultat de la météo
  formatWeatherResult(data, lang = "fr") {
    // Convertir la direction du vent en points cardinaux
    const getWindDirection = (deg) => {
      const directions = ["N", "NE", "E", "SE", "S", "SO", "O", "NO"];
      const index = Math.round(deg / 45) % 8;
      return directions[index];
    };

    // Obtenir l'émoji de la météo
    const weatherEmoji = this.getWeatherEmoji(data.weather[0].id);

    // Formater la description
    const description = data.weather[0].description;
    const formattedDescription =
      description.charAt(0).toUpperCase() + description.slice(1);

    // Formater les données en français ou en anglais
    if (lang === "fr") {
      return {
        location: `${data.name}, ${data.sys.country}`,
        description: `${weatherEmoji} ${formattedDescription}`,
        temperature: `🌡️ Température: ${Math.round(
          data.main.temp
        )}°C (ressentie: ${Math.round(data.main.feels_like)}°C)`,
        humidity: `💧 Humidité: ${data.main.humidity}%`,
        wind: `💨 Vent: ${Math.round(
          data.wind.speed * 3.6
        )} km/h ${getWindDirection(data.wind.deg)}`,
        pressure: `🔄 Pression: ${data.main.pressure} hPa`,
        sunrise: `🌅 Lever du soleil: ${new Date(
          data.sys.sunrise * 1000
        ).toLocaleTimeString("fr-FR", { hour: "2-digit", minute: "2-digit" })}`,
        sunset: `🌇 Coucher du soleil: ${new Date(
          data.sys.sunset * 1000
        ).toLocaleTimeString("fr-FR", { hour: "2-digit", minute: "2-digit" })}`,
      };
    } else {
      return {
        location: `${data.name}, ${data.sys.country}`,
        description: `${weatherEmoji} ${formattedDescription}`,
        temperature: `🌡️ Temperature: ${Math.round(
          data.main.temp
        )}°C (feels like: ${Math.round(data.main.feels_like)}°C)`,
        humidity: `💧 Humidity: ${data.main.humidity}%`,
        wind: `💨 Wind: ${Math.round(
          data.wind.speed * 3.6
        )} km/h ${getWindDirection(data.wind.deg)}`,
        pressure: `🔄 Pressure: ${data.main.pressure} hPa`,
        sunrise: `🌅 Sunrise: ${new Date(
          data.sys.sunrise * 1000
        ).toLocaleTimeString("en-US", { hour: "2-digit", minute: "2-digit" })}`,
        sunset: `🌇 Sunset: ${new Date(
          data.sys.sunset * 1000
        ).toLocaleTimeString("en-US", { hour: "2-digit", minute: "2-digit" })}`,
      };
    }
  }
}

// Créer et exporter une instance avec la clé API
// Utiliser une variable d'environnement pour la clé API
const weatherApi = new WeatherApi(process.env.OPENWEATHERMAP_API_KEY);
module.exports = weatherApi;
```

### Commande Météo pour Discord

```javascript
// src/commands/meteo.js (Discord)
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const weatherApi = require("../apis/weatherApi");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("meteo")
    .setDescription("Affiche la météo pour une ville")
    .addStringOption((option) =>
      option
        .setName("ville")
        .setDescription("Nom de la ville")
        .setRequired(true)
    ),

  async execute(interaction) {
    // Obtenir le nom de la ville
    const city = interaction.options.getString("ville");

    try {
      // Déférer la réponse pendant que nous appelons l'API
      await interaction.deferReply();

      // Appeler l'API météo
      const weatherData = await weatherApi.getCurrentWeather(city);

      // Formater les données
      const formattedData = weatherApi.formatWeatherResult(weatherData);

      // Créer l'embed
      const embed = new EmbedBuilder()
        .setColor(0x3498db)
        .setTitle(`Météo à ${formattedData.location}`)
        .setDescription(formattedData.description)
        .addFields(
          {
            name: "Température",
            value: formattedData.temperature,
            inline: false,
          },
          { name: "Humidité", value: formattedData.humidity, inline: true },
          { name: "Vent", value: formattedData.wind, inline: true },
          { name: "Pression", value: formattedData.pressure, inline: true },
          {
            name: "Lever du soleil",
            value: formattedData.sunrise,
            inline: true,
          },
          {
            name: "Coucher du soleil",
            value: formattedData.sunset,
            inline: true,
          }
        )
        .setThumbnail(
          `http://openweathermap.org/img/wn/${weatherData.weather[0].icon}@2x.png`
        )
        .setFooter({ text: "Données fournies par OpenWeatherMap" })
        .setTimestamp();

      // Envoyer l'embed
      await interaction.editReply({ embeds: [embed] });
    } catch (error) {
      console.error("Erreur lors de la récupération des données météo:", error);

      // Vérifier si l'erreur est due à une ville non trouvée
      if (error.message.includes("404")) {
        await interaction.editReply(
          `Je n'ai pas trouvé de données météo pour la ville "${city}". Vérifiez l'orthographe ou essayez une ville plus grande à proximité.`
        );
      } else {
        await interaction.editReply(
          "Une erreur s'est produite lors de la récupération des données météo. Veuillez réessayer plus tard."
        );
      }
    }
  },
};
```

### Commande Météo pour Twitch

```javascript
// src/commands/meteo.js (Twitch)
const weatherApi = require("../apis/weatherApi");

module.exports = {
  name: "meteo",
  description: "Affiche la météo pour une ville",
  execute: async (client, channel, tags, args) => {
    // Vérifier si une ville a été spécifiée
    if (args.length === 0) {
      return client.say(
        channel,
        `@${tags.username}, veuillez spécifier une ville. Exemple: !meteo Paris`
      );
    }

    // Obtenir le nom de la ville (peut contenir des espaces)
    const city = args.join(" ");

    try {
      // Appeler l'API météo
      const weatherData = await weatherApi.getCurrentWeather(city);

      // Formater les données
      const formattedData = weatherApi.formatWeatherResult(weatherData);

      // Construire le message
      const message = [
        `Météo à ${formattedData.location}: ${formattedData.description}`,
        `${formattedData.temperature} | ${formattedData.humidity} | ${formattedData.wind}`,
      ].join(" — ");

      // Envoyer le message
      client.say(channel, message);
    } catch (error) {
      console.error("Erreur lors de la récupération des données météo:", error);

      // Vérifier si l'erreur est due à une ville non trouvée
      if (error.message.includes("404")) {
        client.say(
          channel,
          `@${tags.username}, je n'ai pas trouvé de données météo pour la ville "${city}". Vérifiez l'orthographe ou essayez une ville plus grande à proximité.`
        );
      } else {
        client.say(
          channel,
          `@${tags.username}, une erreur s'est produite lors de la récupération des données météo. Veuillez réessayer plus tard.`
        );
      }
    }
  },
};
```

## Planification de tâches et rappels

La planification de tâches est utile pour automatiser certaines actions à des moments précis.

### Utilitaire de planification

```javascript
// src/utils/scheduler.js
class Scheduler {
  constructor() {
    this.tasks = new Map();
    this.intervals = new Map();
  }

  // Planifier une tâche unique
  schedule(taskId, callback, delay) {
    // Annuler la tâche existante si elle existe déjà
    this.cancel(taskId);

    // Créer le timer et stocker sa référence
    const timer = setTimeout(() => {
      this.tasks.delete(taskId);
      callback();
    }, delay);

    this.tasks.set(taskId, timer);

    return {
      id: taskId,
      executeAt: new Date(Date.now() + delay),
      cancel: () => this.cancel(taskId),
    };
  }

  // Planifier une tâche récurrente
  scheduleInterval(taskId, callback, interval) {
    // Annuler l'intervalle existant s'il existe déjà
    this.cancelInterval(taskId);

    // Créer l'intervalle et stocker sa référence
    const timer = setInterval(callback, interval);

    this.intervals.set(taskId, timer);

    return {
      id: taskId,
      interval,
      cancel: () => this.cancelInterval(taskId),
    };
  }

  // Annuler une tâche unique
  cancel(taskId) {
    if (this.tasks.has(taskId)) {
      clearTimeout(this.tasks.get(taskId));
      this.tasks.delete(taskId);
      return true;
    }
    return false;
  }

  // Annuler une tâche récurrente
  cancelInterval(taskId) {
    if (this.intervals.has(taskId)) {
      clearInterval(this.intervals.get(taskId));
      this.intervals.delete(taskId);
      return true;
    }
    return false;
  }

  // Obtenir toutes les tâches planifiées
  getTasks() {
    return Array.from(this.tasks.keys()).map((taskId) => ({
      id: taskId,
      executeAt: new Date(
        this.tasks.get(taskId)._idleStart + this.tasks.get(taskId)._idleTimeout
      ),
    }));
  }

  // Obtenir tous les intervalles planifiés
  getIntervals() {
    return Array.from(this.intervals.keys()).map((taskId) => ({
      id: taskId,
      interval: this.intervals.get(taskId)._repeat,
    }));
  }

  // Annuler toutes les tâches et intervalles
  cancelAll() {
    // Annuler toutes les tâches uniques
    for (const taskId of this.tasks.keys()) {
      this.cancel(taskId);
    }

    // Annuler tous les intervalles
    for (const taskId of this.intervals.keys()) {
      this.cancelInterval(taskId);
    }
  }
}

// Créer et exporter une instance unique
const scheduler = new Scheduler();
module.exports = scheduler;
```

### Commande de rappel pour Discord

```javascript
// src/commands/rappel.js (Discord)
const { SlashCommandBuilder, EmbedBuilder } = require("discord.js");
const scheduler = require("../utils/scheduler");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("rappel")
    .setDescription("Crée un rappel")
    .addStringOption((option) =>
      option
        .setName("message")
        .setDescription("Le message du rappel")
        .setRequired(true)
    )
    .addIntegerOption((option) =>
      option
        .setName("minutes")
        .setDescription("Délai en minutes")
        .setMinValue(1)
        .setMaxValue(1440) // Maximum 24 heures
        .setRequired(true)
    ),

  async execute(interaction) {
    // Obtenir les options
    const message = interaction.options.getString("message");
    const minutes = interaction.options.getInteger("minutes");

    // Calculer le délai en millisecondes
    const delay = minutes * 60 * 1000;

    // Générer un ID unique pour le rappel
    const taskId = `reminder_${interaction.user.id}_${Date.now()}`;

    // Planifier le rappel
    const reminder = scheduler.schedule(
      taskId,
      async () => {
        try {
          // Envoyer le rappel
          const reminderEmbed = new EmbedBuilder()
            .setColor("#FF9900")
            .setTitle("⏰ Rappel")
            .setDescription(message)
            .setFooter({ text: `Rappel demandé il y a ${minutes} minute(s)` })
            .setTimestamp();

          await interaction.user
            .send({ embeds: [reminderEmbed] })
            .catch((error) => {
              // L'utilisateur a peut-être désactivé les DM
              console.log(
                `Impossible d'envoyer un rappel à ${interaction.user.tag}:`,
                error
              );

              // Essayer d'envoyer dans le canal d'origine si possible
              interaction.channel
                .send({
                  content: `<@${interaction.user.id}>, voici votre rappel (je n'ai pas pu vous envoyer de message privé) :`,
                  embeds: [reminderEmbed],
                })
                .catch(() => {
                  console.error(
                    `Impossible d'envoyer le rappel à ${interaction.user.tag} dans aucun canal`
                  );
                });
            });
        } catch (error) {
          console.error("Erreur lors de l'envoi du rappel:", error);
        }
      },
      delay
    );

    // Calculer l'heure d'exécution
    const executeTime = new Date(Date.now() + delay);
    const formattedTime = executeTime.toLocaleTimeString();
    const formattedDate = executeTime.toLocaleDateString();

    // Répondre à l'interaction
    await interaction.reply({
      content: `✅ Rappel créé ! Je vous rappellerai "${message}" le ${formattedDate} à ${formattedTime} (dans ${minutes} minute(s)).`,
      ephemeral: true,
    });
  },
};
```

### Commande d'annonces programmées pour Discord

```javascript
// src/commands/annonce.js (Discord)
const {
  SlashCommandBuilder,
  EmbedBuilder,
  PermissionFlagsBits,
} = require("discord.js");
const scheduler = require("../utils/scheduler");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("annonce")
    .setDescription("Programme une annonce")
    .addSubcommand((subcommand) =>
      subcommand
        .setName("programmer")
        .setDescription("Programme une annonce unique")
        .addStringOption((option) =>
          option
            .setName("titre")
            .setDescription("Titre de l'annonce")
            .setRequired(true)
        )
        .addStringOption((option) =>
          option
            .setName("message")
            .setDescription("Contenu de l'annonce")
            .setRequired(true)
        )
        .addChannelOption((option) =>
          option
            .setName("canal")
            .setDescription("Canal où envoyer l'annonce")
            .setRequired(true)
        )
        .addIntegerOption((option) =>
          option
            .setName("minutes")
            .setDescription("Délai en minutes")
            .setMinValue(1)
            .setMaxValue(10080) // Maximum 1 semaine
            .setRequired(true)
        )
        .addStringOption((option) =>
          option
            .setName("couleur")
            .setDescription("Couleur de l'annonce (format hexadécimal)")
            .setRequired(false)
        )
    )
    .addSubcommand((subcommand) =>
      subcommand
        .setName("recurrente")
        .setDescription("Programme une annonce récurrente")
        .addStringOption((option) =>
          option
            .setName("titre")
            .setDescription("Titre de l'annonce")
            .setRequired(true)
        )
        .addStringOption((option) =>
          option
            .setName("message")
            .setDescription("Contenu de l'annonce")
            .setRequired(true)
        )
        .addChannelOption((option) =>
          option
            .setName("canal")
            .setDescription("Canal où envoyer l'annonce")
            .setRequired(true)
        )
        .addIntegerOption((option) =>
          option
            .setName("intervalle")
            .setDescription("Intervalle en minutes")
            .setMinValue(10)
            .setMaxValue(10080) // Maximum 1 semaine
            .setRequired(true)
        )
        .addStringOption((option) =>
          option
            .setName("couleur")
            .setDescription("Couleur de l'annonce (format hexadécimal)")
            .setRequired(false)
        )
    )
    .addSubcommand((subcommand) =>
      subcommand
        .setName("annuler")
        .setDescription("Annule une annonce programmée")
        .addStringOption((option) =>
          option
            .setName("id")
            .setDescription("ID de l'annonce à annuler")
            .setRequired(true)
        )
    )
    .addSubcommand((subcommand) =>
      subcommand
        .setName("liste")
        .setDescription("Liste toutes les annonces programmées")
    )
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),

  async execute(interaction) {
    const subcommand = interaction.options.getSubcommand();

    if (subcommand === "programmer") {
      // Obtenir les options
      const title = interaction.options.getString("titre");
      const message = interaction.options.getString("message");
      const channel = interaction.options.getChannel("canal");
      const minutes = interaction.options.getInteger("minutes");
      const color = interaction.options.getString("couleur") || "#3498DB";

      // Vérifier si le canal est un canal de texte
      if (!channel.isTextBased()) {
        return interaction.reply({
          content: "Vous devez sélectionner un canal de texte.",
          ephemeral: true,
        });
      }

      // Vérifier si le bot a les permissions d'envoyer des messages dans ce canal
      const permissions = channel.permissionsFor(interaction.client.user);
      if (
        !permissions.has(PermissionFlagsBits.SendMessages) ||
        !permissions.has(PermissionFlagsBits.ViewChannel)
      ) {
        return interaction.reply({
          content:
            "Je n'ai pas les permissions nécessaires pour envoyer des messages dans ce canal.",
          ephemeral: true,
        });
      }

      // Vérifier le format de la couleur
      if (!/^#[0-9A-F]{6}$/i.test(color)) {
        return interaction.reply({
          content: "La couleur doit être au format hexadécimal (ex: #3498DB).",
          ephemeral: true,
        });
      }

      // Calculer le délai en millisecondes
      const delay = minutes * 60 * 1000;

      // Générer un ID unique pour l'annonce
      const taskId = `announcement_${interaction.guild.id}_${Date.now()}`;

      // Créer l'embed de l'annonce
      const announcementEmbed = new EmbedBuilder()
        .setColor(color)
        .setTitle(title)
        .setDescription(message)
        .setFooter({ text: `Annonce programmée par ${interaction.user.tag}` })
        .setTimestamp();

      // Planifier l'annonce
      const announcement = scheduler.schedule(
        taskId,
        async () => {
          try {
            await channel.send({ embeds: [announcementEmbed] });
            console.log(
              `Annonce "${title}" envoyée dans le canal #${channel.name}`
            );
          } catch (error) {
            console.error("Erreur lors de l'envoi de l'annonce:", error);
          }
        },
        delay
      );

      // Calculer l'heure d'exécution
      const executeTime = new Date(Date.now() + delay);
      const formattedTime = executeTime.toLocaleTimeString();
      const formattedDate = executeTime.toLocaleDateString();

      // Répondre à l'interaction
      await interaction.reply({
        content: `✅ Annonce programmée ! L'annonce "${title}" sera envoyée dans <#${channel.id}> le ${formattedDate} à ${formattedTime} (dans ${minutes} minute(s)).\n\nID de l'annonce: \`${taskId}\``,
        ephemeral: true,
      });
    } else if (subcommand === "recurrente") {
      // Obtenir les options
      const title = interaction.options.getString("titre");
      const message = interaction.options.getString("message");
      const channel = interaction.options.getChannel("canal");
      const interval = interaction.options.getInteger("intervalle");
      const color = interaction.options.getString("couleur") || "#3498DB";

      // Vérifier si le canal est un canal de texte
      if (!channel.isTextBased()) {
        return interaction.reply({
          content: "Vous devez sélectionner un canal de texte.",
          ephemeral: true,
        });
      }

      // Vérifier si le bot a les permissions d'envoyer des messages dans ce canal
      const permissions = channel.permissionsFor(interaction.client.user);
      if (
        !permissions.has(PermissionFlagsBits.SendMessages) ||
        !permissions.has(PermissionFlagsBits.ViewChannel)
      ) {
        return interaction.reply({
          content:
            "Je n'ai pas les permissions nécessaires pour envoyer des messages dans ce canal.",
          ephemeral: true,
        });
      }

      // Vérifier le format de la couleur
      if (!/^#[0-9A-F]{6}$/i.test(color)) {
        return interaction.reply({
          content: "La couleur doit être au format hexadécimal (ex: #3498DB).",
          ephemeral: true,
        });
      }

      // Calculer l'intervalle en millisecondes
      const intervalMs = interval * 60 * 1000;

      // Générer un ID unique pour l'annonce
      const taskId = `recurring_${interaction.guild.id}_${Date.now()}`;

      // Créer l'embed de l'annonce
      const announcementEmbed = new EmbedBuilder()
        .setColor(color)
        .setTitle(title)
        .setDescription(message)
        .setFooter({
          text: `Annonce récurrente programmée par ${interaction.user.tag}`,
        })
        .setTimestamp();

      // Planifier l'annonce récurrente
      const announcement = scheduler.scheduleInterval(
        taskId,
        async () => {
          try {
            // Mettre à jour le timestamp à chaque envoi
            announcementEmbed.setTimestamp();

            await channel.send({ embeds: [announcementEmbed] });
            console.log(
              `Annonce récurrente "${title}" envoyée dans le canal #${channel.name}`
            );
          } catch (error) {
            console.error(
              "Erreur lors de l'envoi de l'annonce récurrente:",
              error
            );
          }
        },
        intervalMs
      );

      // Répondre à l'interaction
      await interaction.reply({
        content: `✅ Annonce récurrente programmée ! L'annonce "${title}" sera envoyée dans <#${channel.id}> toutes les ${interval} minute(s).\n\nID de l'annonce: \`${taskId}\``,
        ephemeral: true,
      });
    } else if (subcommand === "annuler") {
      // Obtenir l'ID de l'annonce
      const taskId = interaction.options.getString("id");

      // Déterminer s'il s'agit d'une annonce unique ou récurrente
      let isCancelled = false;

      // Essayer d'annuler en tant qu'annonce unique
      if (scheduler.cancel(taskId)) {
        isCancelled = true;
      }
      // Sinon, essayer d'annuler en tant qu'annonce récurrente
      else if (scheduler.cancelInterval(taskId)) {
        isCancelled = true;
      }

      // Répondre en fonction du résultat
      if (isCancelled) {
        await interaction.reply({
          content: `✅ L'annonce avec l'ID \`${taskId}\` a été annulée avec succès.`,
          ephemeral: true,
        });
      } else {
        await interaction.reply({
          content: `❌ Aucune annonce trouvée avec l'ID \`${taskId}\`.`,
          ephemeral: true,
        });
      }
    } else if (subcommand === "liste") {
      // Obtenir toutes les annonces
      const tasks = scheduler.getTasks();
      const intervals = scheduler.getIntervals();

      // Filtrer pour ne garder que les annonces concernant ce serveur
      const serverTasks = tasks.filter((task) =>
        task.id.includes(`announcement_${interaction.guild.id}`)
      );
      const serverIntervals = intervals.filter((interval) =>
        interval.id.includes(`recurring_${interaction.guild.id}`)
      );

      // Créer l'embed
      const embed = new EmbedBuilder()
        .setColor("#3498DB")
        .setTitle("📅 Annonces programmées")
        .setDescription(
          `${serverTasks.length} annonce(s) unique(s) et ${serverIntervals.length} annonce(s) récurrente(s) programmée(s).`
        );

      // Ajouter les annonces uniques
      if (serverTasks.length > 0) {
        let tasksText = "";

        for (const task of serverTasks) {
          const formattedTime = task.executeAt.toLocaleTimeString();
          const formattedDate = task.executeAt.toLocaleDateString();
          tasksText += `• \`${task.id}\` - ${formattedDate} à ${formattedTime}\n`;
        }

        embed.addFields({ name: "📌 Annonces uniques", value: tasksText });
      }

      // Ajouter les annonces récurrentes
      if (serverIntervals.length > 0) {
        let intervalsText = "";

        for (const interval of serverIntervals) {
          const minutes = Math.floor(interval.interval / 60000);
          intervalsText += `• \`${interval.id}\` - Toutes les ${minutes} minute(s)\n`;
        }

        embed.addFields({
          name: "🔄 Annonces récurrentes",
          value: intervalsText,
        });
      }

      // Ajouter un guide sur la commande d'annulation
      embed.addFields({
        name: "ℹ️ Comment annuler",
        value:
          "Pour annuler une annonce, utilisez la commande `/annonce annuler` avec l'ID correspondant.",
      });

      // Répondre à l'interaction
      await interaction.reply({
        embeds: [embed],
        ephemeral: true,
      });
    }
  },
};
```

## Systèmes de sondages et votes avancés

Les sondages sont un excellent moyen d'engager votre communauté. Voici une implémentation avancée :

### Classe de gestion des sondages

```javascript
// src/utils/pollManager.js
const {
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
} = require("discord.js");

class PollManager {
  constructor() {
    // Stocker les sondages actifs
    // Structure: Map<pollId, pollData>
    this.activePolls = new Map();

    // Emojis pour les options
    this.optionEmojis = [
      "1️⃣",
      "2️⃣",
      "3️⃣",
      "4️⃣",
      "5️⃣",
      "6️⃣",
      "7️⃣",
      "8️⃣",
      "9️⃣",
      "🔟",
    ];

    // Durées prédéfinies en millisecondes
    this.durations = {
      "1m": 60 * 1000,
      "5m": 5 * 60 * 1000,
      "15m": 15 * 60 * 1000,
      "30m": 30 * 60 * 1000,
      "1h": 60 * 60 * 1000,
      "6h": 6 * 60 * 60 * 1000,
      "12h": 12 * 60 * 60 * 1000,
      "1d": 24 * 60 * 60 * 1000,
      "3d": 3 * 24 * 60 * 60 * 1000,
      "1w": 7 * 24 * 60 * 60 * 1000,
    };
  }

  // Créer un nouveau sondage
  async createPoll(
    interaction,
    question,
    options,
    duration,
    isAnonymous = false,
    allowMultiple = false
  ) {
    // Vérifier les limites
    if (options.length < 2) {
      throw new Error("Un sondage doit avoir au moins 2 options.");
    }

    if (options.length > this.optionEmojis.length) {
      throw new Error(
        `Un sondage ne peut pas avoir plus de ${this.optionEmojis.length} options.`
      );
    }

    // Générer un ID unique pour le sondage
    const pollId = `poll_${Date.now()}_${Math.floor(Math.random() * 1000)}`;

    // Créer la structure du sondage
    const poll = {
      id: pollId,
      question,
      options: options.map((option, index) => ({
        id: `option_${index}`,
        text: option,
        emoji: this.optionEmojis[index],
        votes: 0,
        voters: [],
      })),
      createdBy: interaction.user.id,
      createdAt: new Date(),
      endsAt: new Date(Date.now() + this.durations[duration]),
      isAnonymous,
      allowMultiple,
      totalVotes: 0,
      voters: new Set(),
      messageId: null,
      channelId: interaction.channelId,
    };

    // Créer l'embed initial
    const embed = this.createPollEmbed(poll);

    // Créer les boutons pour voter
    const actionRows = this.createPollButtons(poll);

    // Envoyer le message du sondage
    const pollMessage = await interaction.reply({
      embeds: [embed],
      components: actionRows,
      fetchReply: true,
    });

    // Stocker l'ID du message
    poll.messageId = pollMessage.id;

    // Enregistrer le sondage
    this.activePolls.set(pollId, poll);

    // Planifier la fin du sondage
    setTimeout(() => this.endPoll(poll.id), this.durations[duration]);

    return poll;
  }

  // Créer l'embed pour le sondage
  createPollEmbed(poll) {
    // Calculer le temps restant
    const now = new Date();
    const timeLeft = poll.endsAt - now;

    let timeLeftText;
    if (timeLeft <= 0) {
      timeLeftText = "Terminé";
    } else {
      const hours = Math.floor(timeLeft / (1000 * 60 * 60));
      const minutes = Math.floor((timeLeft % (1000 * 60 * 60)) / (1000 * 60));

      if (hours > 0) {
        timeLeftText = `${hours}h ${minutes}m restantes`;
      } else {
        timeLeftText = `${minutes}m restantes`;
      }
    }

    // Créer le texte des résultats
    let resultsText = "";

    for (const option of poll.options) {
      const percentage =
        poll.totalVotes > 0
          ? Math.round((option.votes / poll.totalVotes) * 100)
          : 0;
      const progressBar = this.createProgressBar(percentage);

      resultsText += `${option.emoji} **${option.text}**\n`;
      resultsText += `${progressBar} ${percentage}% (${option.votes} vote${
        option.votes !== 1 ? "s" : ""
      })\n\n`;
    }

    // Créer l'embed
    const embed = new EmbedBuilder()
      .setColor("#3498DB")
      .setTitle(`📊 Sondage: ${poll.question}`)
      .setDescription(resultsText)
      .addFields({ name: "Temps restant", value: timeLeftText })
      .setFooter({
        text: `${
          poll.isAnonymous ? "🔒 Sondage anonyme" : "👁️ Sondage public"
        } • ${
          poll.allowMultiple
            ? "Votes multiples autorisés"
            : "Un seul vote par personne"
        } • Total: ${poll.totalVotes} vote${poll.totalVotes !== 1 ? "s" : ""}`,
      })
      .setTimestamp();

    return embed;
  }

  // Créer les boutons pour le sondage
  createPollButtons(poll) {
    const actionRows = [];
    let currentRow = new ActionRowBuilder();
    let buttonCount = 0;

    // Créer un bouton pour chaque option
    for (const option of poll.options) {
      const button = new ButtonBuilder()
        .setCustomId(`vote_${poll.id}_${option.id}`)
        .setLabel(option.text)
        .setEmoji(option.emoji)
        .setStyle(ButtonStyle.Primary);

      currentRow.addComponents(button);
      buttonCount++;

      // Maximum 5 boutons par ligne
      if (buttonCount % 5 === 0 || buttonCount === poll.options.length) {
        actionRows.push(currentRow);
        currentRow = new ActionRowBuilder();
      }
    }

    return actionRows;
  }

  // Gérer un vote
  async handleVote(interaction, pollId, optionId) {
    // Vérifier si le sondage existe
    if (!this.activePolls.has(pollId)) {
      return await interaction.reply({
        content: "Ce sondage n'existe plus ou est terminé.",
        ephemeral: true,
      });
    }

    const poll = this.activePolls.get(pollId);
    const option = poll.options.find((opt) => opt.id === optionId);

    if (!option) {
      return await interaction.reply({
        content: "Option de vote invalide.",
        ephemeral: true,
      });
    }

    // Vérifier si le sondage est terminé
    if (new Date() > poll.endsAt) {
      return await interaction.reply({
        content: "Ce sondage est terminé.",
        ephemeral: true,
      });
    }

    const userId = interaction.user.id;

    // Vérifier si l'utilisateur a déjà voté
    if (!poll.allowMultiple && poll.voters.has(userId)) {
      // Trouver l'option pour laquelle l'utilisateur a voté
      const previousOption = poll.options.find((opt) =>
        opt.voters.includes(userId)
      );

      if (previousOption) {
        // Si l'utilisateur vote pour la même option, annuler son vote
        if (previousOption.id === optionId) {
          // Annuler le vote
          previousOption.votes--;
          previousOption.voters = previousOption.voters.filter(
            (id) => id !== userId
          );
          poll.totalVotes--;
          poll.voters.delete(userId);

          // Mettre à jour l'embed
          const updatedEmbed = this.createPollEmbed(poll);
          await interaction.update({ embeds: [updatedEmbed] });

          return await interaction.followUp({
            content: "Votre vote a été annulé.",
            ephemeral: true,
          });
        } else {
          // Si l'utilisateur vote pour une option différente, changer son vote
          previousOption.votes--;
          previousOption.voters = previousOption.voters.filter(
            (id) => id !== userId
          );

          option.votes++;
          option.voters.push(userId);

          // Mettre à jour l'embed
          const updatedEmbed = this.createPollEmbed(poll);
          await interaction.update({ embeds: [updatedEmbed] });

          return await interaction.followUp({
            content: `Votre vote a été changé pour "${option.text}".`,
            ephemeral: true,
          });
        }
      }
    } else if (poll.allowMultiple && option.voters.includes(userId)) {
      // Si les votes multiples sont autorisés et l'utilisateur a déjà voté pour cette option, annuler son vote
      option.votes--;
      option.voters = option.voters.filter((id) => id !== userId);
      poll.totalVotes--;

      // Vérifier si l'utilisateur a encore des votes
      let hasOtherVotes = false;
      for (const opt of poll.options) {
        if (opt.voters.includes(userId)) {
          hasOtherVotes = true;
          break;
        }
      }

      if (!hasOtherVotes) {
        poll.voters.delete(userId);
      }

      // Mettre à jour l'embed
      const updatedEmbed = this.createPollEmbed(poll);
      await interaction.update({ embeds: [updatedEmbed] });

      return await interaction.followUp({
        content: `Votre vote pour "${option.text}" a été annulé.`,
        ephemeral: true,
      });
    } else {
      // Ajouter le vote
      option.votes++;
      option.voters.push(userId);
      poll.totalVotes++;
      poll.voters.add(userId);

      // Mettre à jour l'embed
      const updatedEmbed = this.createPollEmbed(poll);
      await interaction.update({ embeds: [updatedEmbed] });

      return await interaction.followUp({
        content: `Vous avez voté pour "${option.text}".`,
        ephemeral: true,
      });
    }
  }

  // Terminer un sondage
  async endPoll(pollId) {
    // Vérifier si le sondage existe
    if (!this.activePolls.has(pollId)) {
      return;
    }

    const poll = this.activePolls.get(pollId);

    // Trouver le(s) option(s) gagnante(s)
    const maxVotes = Math.max(...poll.options.map((option) => option.votes));
    const winners = poll.options.filter((option) => option.votes === maxVotes);

    // Créer l'embed des résultats
    const embed = this.createPollEmbed(poll);

    // Ajouter le(s) gagnant(s) à l'embed
    if (poll.totalVotes > 0) {
      let winnersText =
        winners.length === 1
          ? `🏆 L'option gagnante est: **${winners[0].text}** avec ${
              winners[0].votes
            } vote${winners[0].votes !== 1 ? "s" : ""} (${Math.round(
              (winners[0].votes / poll.totalVotes) * 100
            )}%).`
          : `🏆 Il y a une égalité entre les options: ${winners
              .map((w) => `**${w.text}**`)
              .join(", ")} avec ${maxVotes} vote${
              maxVotes !== 1 ? "s" : ""
            } chacune (${Math.round((maxVotes / poll.totalVotes) * 100)}%).`;

      embed.addFields({ name: "Résultat", value: winnersText });
    } else {
      embed.addFields({
        name: "Résultat",
        value: "Aucun vote n'a été enregistré pour ce sondage.",
      });
    }

    // Mettre à jour le message du sondage
    try {
      // Obtenir le client Discord
      const client = require("../index").client;

      // Obtenir le canal
      const channel = client.channels.cache.get(poll.channelId);

      if (channel) {
        // Obtenir le message
        const message = await channel.messages.fetch(poll.messageId);

        if (message) {
          // Mettre à jour le message
          await message.edit({
            embeds: [embed],
            components: [], // Supprimer les boutons
          });

          // Envoyer un message pour indiquer que le sondage est terminé
          await channel.send({
            content: `📊 Le sondage "${poll.question}" est terminé !`,
            embeds: [embed],
          });
        }
      }
    } catch (error) {
      console.error(
        "Erreur lors de la mise à jour du message de sondage:",
        error
      );
    }

    // Supprimer le sondage de la liste des sondages actifs
    this.activePolls.delete(pollId);
  }

  // Créer une barre de progression
  createProgressBar(percentage, length = 15) {
    const filledLength = Math.round((percentage / 100) * length);
    const emptyLength = length - filledLength;

    return "█".repeat(filledLength) + "░".repeat(emptyLength);
  }

  // Obtenir un sondage par son ID
  getPoll(pollId) {
    return this.activePolls.get(pollId);
  }

  // Vérifier si un sondage existe
  hasPoll(pollId) {
    return this.activePolls.has(pollId);
  }

  // Obtenir tous les sondages
  getAllPolls() {
    return Array.from(this.activePolls.values());
  }
}

// Créer et exporter une instance unique
const pollManager = new PollManager();
module.exports = pollManager;
```

### Commande de sondage pour Discord

```javascript
// src/commands/sondage.js (Discord)
const { SlashCommandBuilder } = require("discord.js");
const pollManager = require("../utils/pollManager");

module.exports = {
  data: new SlashCommandBuilder()
    .setName("sondage")
    .setDescription("Crée un sondage")
    .addStringOption((option) =>
      option
        .setName("question")
        .setDescription("La question du sondage")
        .setRequired(true)
    )
    .addStringOption((option) =>
      option.setName("option1").setDescription("Option 1").setRequired(true)
    )
    .addStringOption((option) =>
      option.setName("option2").setDescription("Option 2").setRequired(true)
    )
    .addStringOption((option) =>
      option.setName("option3").setDescription("Option 3").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option4").setDescription("Option 4").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option5").setDescription("Option 5").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option6").setDescription("Option 6").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option7").setDescription("Option 7").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option8").setDescription("Option 8").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option9").setDescription("Option 9").setRequired(false)
    )
    .addStringOption((option) =>
      option.setName("option10").setDescription("Option 10").setRequired(false)
    )
    .addStringOption((option) =>
      option
        .setName("duree")
        .setDescription("Durée du sondage")
        .setRequired(true)
        .addChoices(
          { name: "1 minute", value: "1m" },
          { name: "5 minutes", value: "5m" },
          { name: "15 minutes", value: "15m" },
          { name: "30 minutes", value: "30m" },
          { name: "1 heure", value: "1h" },
          { name: "6 heures", value: "6h" },
          { name: "12 heures", value: "12h" },
          { name: "1 jour", value: "1d" },
          { name: "3 jours", value: "3d" },
          { name: "1 semaine", value: "1w" }
        )
    )
    .addBooleanOption((option) =>
      option
        .setName("anonyme")
        .setDescription("Les votes sont-ils anonymes?")
        .setRequired(false)
    )
    .addBooleanOption((option) =>
      option
        .setName("votes_multiples")
        .setDescription("Autoriser les votes multiples?")
        .setRequired(false)
    ),

  async execute(interaction) {
    try {
      // Obtenir les options
      const question = interaction.options.getString("question");
      const duration = interaction.options.getString("duree");
      const isAnonymous = interaction.options.getBoolean("anonyme") || false;
      const allowMultiple =
        interaction.options.getBoolean("votes_multiples") || false;

      // Récupérer toutes les options
      const options = [];
      for (let i = 1; i <= 10; i++) {
        const option = interaction.options.getString(`option${i}`);
        if (option) options.push(option);
      }

      // Créer le sondage
      await pollManager.createPoll(
        interaction,
        question,
        options,
        duration,
        isAnonymous,
        allowMultiple
      );
    } catch (error) {
      console.error("Erreur lors de la création du sondage:", error);

      // Répondre avec l'erreur
      if (!interaction.replied) {
        await interaction.reply({
          content: `Une erreur s'est produite: ${error.message}`,
          ephemeral: true,
        });
      } else {
        await interaction.followUp({
          content: `Une erreur s'est produite: ${error.message}`,
          ephemeral: true,
        });
      }
    }
  },
};
```

### Gestionnaire d'interactions pour les votes

```javascript
// src/events/interactionCreate.js (mise à jour pour Discord)
const pollManager = require("../utils/pollManager");

module.exports = {
  name: "interactionCreate",
  once: false,
  async execute(interaction) {
    // Gérer les interactions de bouton
    if (interaction.isButton()) {
      // Vérifier si c'est un vote pour un sondage
      if (interaction.customId.startsWith("vote_")) {
        // Extraire l'ID du sondage et de l'option
        const [, pollId, optionId] = interaction.customId.split("_");

        // Traiter le vote
        await pollManager.handleVote(interaction, pollId, optionId);
        return;
      }

      // Autres interactions de bouton...
    }

    // Gérer les commandes slash
    if (interaction.isChatInputCommand()) {
      // Le reste du code pour les commandes slash...
    }
  },
};
```

## Conclusion

Dans ce chapitre, nous avons exploré des fonctionnalités avancées pour rendre vos bots Discord et Twitch plus puissants et utiles :

1. **Modération automatique** - Filtrage du contenu, avertissements et sanctions
2. **Intégration d'API externes** - Utilisation d'API comme OpenWeatherMap pour enrichir votre bot
3. **Planification de tâches** - Rappels personnels et annonces programmées
4. **Sondages avancés** - Système de vote interactif avec boutons Discord

Ces fonctionnalités peuvent considérablement améliorer l'expérience utilisateur et automatiser des tâches complexes pour les administrateurs et modérateurs. N'hésitez pas à les adapter à vos besoins spécifiques et à explorer d'autres idées innovantes pour vos bots.

Dans le prochain chapitre, nous verrons comment déployer et maintenir vos bots en production, avec un focus sur la fiabilité, la surveillance et l'évolutivité.

## Ressources supplémentaires

- [Discord.js Guide - Boutons](https://discordjs.guide/interactions/buttons.html)
- [Discord Developer Portal](https://discord.com/developers/docs/intro)
- [Twitch Developer Documentation](https://dev.twitch.tv/docs/)
- [OpenWeatherMap API Documentation](https://openweathermap.org/api)
- [Node.js Scheduling](https://nodejs.org/en/docs/guides/timers-in-node/)
