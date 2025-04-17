# Chapitre 9 - Conception d'un tableau de bord web pour votre bot

Ce chapitre traite de la conception et du développement d'un tableau de bord web complet pour l'administration de vos bots Discord et Twitch. Un tableau de bord administratif offre plusieurs avantages significatifs, notamment la possibilité de gérer votre bot sans recourir à des commandes en ligne, de visualiser des statistiques importantes, et de permettre aux administrateurs de serveur de personnaliser le comportement du bot.

## Avantages d'un tableau de bord web

L'implémentation d'un tableau de bord web pour votre bot présente de nombreux avantages :

1. **Gestion simplifiée** : Permet aux utilisateurs non techniques de configurer le bot sans connaître les commandes spécifiques.
2. **Visualisation des données** : Offre une représentation visuelle des statistiques et de l'activité du bot.
3. **Contrôle précis** : Permet des configurations détaillées qui seraient trop complexes via des commandes.
4. **Image professionnelle** : Donne une impression de qualité et de sérieux à votre bot.
5. **Accès sécurisé** : Implémente des niveaux d'autorisation et d'authentification pour la gestion du bot.

## Conception de l'architecture

Notre tableau de bord suivra une architecture moderne basée sur :

1. **Backend** : Node.js avec Express.js
2. **Base de données** : Celle que nous avons déjà configurée (MongoDB ou SQLite)
3. **Frontend** : React.js avec des composants Bootstrap ou Material-UI
4. **Authentification** : OAuth2 avec Discord et/ou Twitch

### Structure du projet

```
dashboard/
├── client/                 # Frontend React
│   ├── public/
│   └── src/
│       ├── components/     # Composants React réutilisables
│       ├── pages/          # Pages du tableau de bord
│       ├── services/       # Services API
│       └── App.js          # Point d'entrée
│
├── server/                 # Backend Express
│   ├── config/             # Configuration
│   ├── controllers/        # Contrôleurs d'API
│   ├── middleware/         # Middleware Express
│   ├── models/             # Modèles de données
│   ├── routes/             # Routes API
│   └── server.js           # Point d'entrée du serveur
│
└── package.json            # Dépendances du projet
```

## Configuration du serveur Express

Commençons par configurer le serveur backend avec Express :

### Installation des dépendances

```bash
mkdir dashboard
cd dashboard
npm init -y
npm install express cors helmet morgan passport passport-discord passport-oauth2 express-session cookie-parser mongoose dotenv jsonwebtoken bcrypt
npm install --save-dev nodemon
```

### Configuration du serveur principal

```javascript
// server/server.js
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const helmet = require("helmet");
const morgan = require("morgan");
const session = require("express-session");
const passport = require("passport");
const path = require("path");
const mongoose = require("mongoose");
const logger = require("../src/utils/logger");

// Importation des routes
const authRoutes = require("./routes/auth");
const apiRoutes = require("./routes/api");
const botRoutes = require("./routes/bot");

// Initialisation de l'application Express
const app = express();
const PORT = process.env.PORT || 3001;

// Configuration de la base de données
mongoose
  .connect(process.env.MONGODB_URI, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => logger.info("Connected to MongoDB"))
  .catch((err) => logger.error("Failed to connect to MongoDB:", err));

// Middleware de sécurité et de journalisation
app.use(helmet());
app.use(cors({ origin: process.env.CLIENT_URL, credentials: true }));
app.use(morgan("dev"));
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Configuration de la session
app.use(
  session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: {
      secure: process.env.NODE_ENV === "production",
      maxAge: 24 * 60 * 60 * 1000, // 24 heures
    },
  })
);

// Initialisation de Passport
app.use(passport.initialize());
app.use(passport.session());

// Configuration des routes
app.use("/auth", authRoutes);
app.use("/api", apiRoutes);
app.use("/bot", botRoutes);

// Servir les fichiers statiques du frontend en production
if (process.env.NODE_ENV === "production") {
  app.use(express.static(path.join(__dirname, "../client/build")));

  app.get("*", (req, res) => {
    res.sendFile(path.join(__dirname, "../client/build", "index.html"));
  });
}

// Gestionnaire d'erreurs global
app.use((err, req, res, next) => {
  logger.error("Server Error:", err);
  res.status(500).json({ error: "Internal Server Error" });
});

// Démarrage du serveur
app.listen(PORT, () => {
  logger.info(`Dashboard server running on port ${PORT}`);
});
```

### Configuration de l'authentification Discord

```javascript
// server/config/passport.js
const passport = require("passport");
const DiscordStrategy = require("passport-discord").Strategy;
const User = require("../models/user");
const logger = require("../../src/utils/logger");

// Définir les informations d'identification
const DISCORD_CLIENT_ID = process.env.DISCORD_CLIENT_ID;
const DISCORD_CLIENT_SECRET = process.env.DISCORD_CLIENT_SECRET;
const CALLBACK_URL = process.env.DISCORD_CALLBACK_URL;

// Sérialisation et désérialisation de l'utilisateur
passport.serializeUser((user, done) => {
  done(null, user.id);
});

passport.deserializeUser(async (id, done) => {
  try {
    const user = await User.findById(id);
    done(null, user);
  } catch (error) {
    logger.error("Error deserializing user:", error);
    done(error, null);
  }
});

// Configuration de la stratégie Discord
passport.use(
  new DiscordStrategy(
    {
      clientID: DISCORD_CLIENT_ID,
      clientSecret: DISCORD_CLIENT_SECRET,
      callbackURL: CALLBACK_URL,
      scope: ["identify", "guilds"],
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        // Rechercher l'utilisateur dans la base de données
        let user = await User.findOne({ discordId: profile.id });

        // Créer un nouvel utilisateur s'il n'existe pas
        if (!user) {
          user = new User({
            discordId: profile.id,
            username: profile.username,
            discriminator: profile.discriminator,
            avatar: profile.avatar,
            guilds: profile.guilds,
            accessToken,
            refreshToken,
          });

          await user.save();
          logger.info(
            `New user registered: ${profile.username}#${profile.discriminator}`
          );
        } else {
          // Mettre à jour les informations existantes
          user.username = profile.username;
          user.discriminator = profile.discriminator;
          user.avatar = profile.avatar;
          user.guilds = profile.guilds;
          user.accessToken = accessToken;
          user.refreshToken = refreshToken;

          await user.save();
          logger.info(
            `User updated: ${profile.username}#${profile.discriminator}`
          );
        }

        return done(null, user);
      } catch (error) {
        logger.error("Error during Discord authentication:", error);
        return done(error, null);
      }
    }
  )
);

module.exports = passport;
```

### Modèle d'utilisateur

```javascript
// server/models/user.js
const mongoose = require("mongoose");

const UserSchema = new mongoose.Schema({
  discordId: {
    type: String,
    required: true,
    unique: true,
  },
  username: {
    type: String,
    required: true,
  },
  discriminator: {
    type: String,
    required: true,
  },
  avatar: {
    type: String,
  },
  guilds: {
    type: Array,
    default: [],
  },
  accessToken: {
    type: String,
    required: true,
  },
  refreshToken: {
    type: String,
  },
  isAdmin: {
    type: Boolean,
    default: false,
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
  lastLogin: {
    type: Date,
    default: Date.now,
  },
});

// Méthode pour vérifier si l'utilisateur est administrateur d'un serveur
UserSchema.methods.isGuildAdmin = function (guildId) {
  const guild = this.guilds.find((g) => g.id === guildId);
  if (!guild) return false;

  // Vérifier si l'utilisateur est le propriétaire ou a des permissions d'administrateur
  return guild.owner || (guild.permissions & 0x8) === 0x8;
};

module.exports = mongoose.model("User", UserSchema);
```

### Routes d'authentification

```javascript
// server/routes/auth.js
const express = require("express");
const passport = require("../config/passport");
const router = express.Router();
const logger = require("../../src/utils/logger");

// Middleware pour vérifier l'authentification
const isAuthenticated = (req, res, next) => {
  if (req.isAuthenticated()) {
    return next();
  }
  res.status(401).json({ error: "Not authenticated" });
};

// Route pour démarrer l'authentification Discord
router.get("/discord", passport.authenticate("discord"));

// Route de callback après l'authentification Discord
router.get(
  "/discord/callback",
  passport.authenticate("discord", {
    failureRedirect: "/auth/failure",
  }),
  (req, res) => {
    // Mise à jour de la dernière connexion
    if (req.user) {
      req.user.lastLogin = new Date();
      req.user
        .save()
        .catch((err) => logger.error("Error updating last login:", err));
    }

    // Redirection vers le tableau de bord
    res.redirect(process.env.CLIENT_URL + "/dashboard");
  }
);

// Route pour vérifier l'état de l'authentification
router.get("/status", (req, res) => {
  if (req.isAuthenticated()) {
    res.json({
      authenticated: true,
      user: {
        id: req.user.id,
        username: req.user.username,
        discriminator: req.user.discriminator,
        avatar: req.user.avatar,
        isAdmin: req.user.isAdmin,
      },
    });
  } else {
    res.json({ authenticated: false });
  }
});

// Route pour obtenir les serveurs de l'utilisateur
router.get("/guilds", isAuthenticated, (req, res) => {
  // Filtrer les serveurs où le bot est présent
  // Ceci nécessite d'avoir une liste des serveurs où le bot est installé
  const userGuilds = req.user.guilds;

  res.json({ guilds: userGuilds });
});

// Route de déconnexion
router.get("/logout", (req, res) => {
  req.logout((err) => {
    if (err) {
      logger.error("Error during logout:", err);
      return res.status(500).json({ error: "Error during logout" });
    }
    res.json({ success: true, message: "Logged out successfully" });
  });
});

// Route d'échec d'authentification
router.get("/failure", (req, res) => {
  res.status(401).json({ error: "Authentication failed" });
});

module.exports = router;
```

### Routes API pour les statistiques et configurations du bot

```javascript
// server/routes/api.js
const express = require("express");
const router = express.Router();
const BotConfig = require("../models/botConfig");
const logger = require("../../src/utils/logger");

// Middleware pour vérifier l'authentification
const isAuthenticated = (req, res, next) => {
  if (req.isAuthenticated()) {
    return next();
  }
  res.status(401).json({ error: "Not authenticated" });
};

// Middleware pour vérifier les permissions d'administrateur sur un serveur
const isGuildAdmin = async (req, res, next) => {
  const { guildId } = req.params;

  if (!req.user.isGuildAdmin(guildId) && !req.user.isAdmin) {
    return res.status(403).json({ error: "Insufficient permissions" });
  }

  next();
};

// Obtenir les statistiques générales du bot
router.get("/stats", isAuthenticated, async (req, res) => {
  try {
    // Collecter les statistiques du bot depuis une source externe
    // Par exemple, on pourrait avoir un service qui expose des métriques
    const stats = {
      guilds: 120, // Exemple de données
      users: 15000,
      commands: 5000,
      uptime: "5d 7h 23m",
      memoryUsage: "156 MB",
    };

    res.json(stats);
  } catch (error) {
    logger.error("Error fetching bot stats:", error);
    res.status(500).json({ error: "Failed to fetch bot statistics" });
  }
});

// Obtenir la configuration d'un serveur
router.get(
  "/guilds/:guildId/config",
  isAuthenticated,
  isGuildAdmin,
  async (req, res) => {
    try {
      const { guildId } = req.params;

      let config = await BotConfig.findOne({ guildId });

      if (!config) {
        // Créer une configuration par défaut si elle n'existe pas
        config = new BotConfig({ guildId });
        await config.save();
      }

      res.json(config);
    } catch (error) {
      logger.error(
        `Error fetching guild config for ${req.params.guildId}:`,
        error
      );
      res.status(500).json({ error: "Failed to fetch guild configuration" });
    }
  }
);

// Mettre à jour la configuration d'un serveur
router.put(
  "/guilds/:guildId/config",
  isAuthenticated,
  isGuildAdmin,
  async (req, res) => {
    try {
      const { guildId } = req.params;
      const updates = req.body;

      // Trouver et mettre à jour la configuration
      const config = await BotConfig.findOneAndUpdate(
        { guildId },
        { $set: updates },
        { new: true, upsert: true }
      );

      // Notifier le bot du changement de configuration
      // Ceci dépendrait de votre implémentation

      res.json(config);
    } catch (error) {
      logger.error(
        `Error updating guild config for ${req.params.guildId}:`,
        error
      );
      res.status(500).json({ error: "Failed to update guild configuration" });
    }
  }
);

// Obtenir l'historique des commandes d'un serveur
router.get(
  "/guilds/:guildId/commands",
  isAuthenticated,
  isGuildAdmin,
  async (req, res) => {
    try {
      const { guildId } = req.params;

      // Recherche dans votre base de données d'historique des commandes
      // Ceci dépendrait de votre implémentation

      const commandHistory = []; // Placeholder pour l'historique des commandes

      res.json(commandHistory);
    } catch (error) {
      logger.error(
        `Error fetching command history for ${req.params.guildId}:`,
        error
      );
      res.status(500).json({ error: "Failed to fetch command history" });
    }
  }
);

module.exports = router;
```

### Modèle de configuration de bot

```javascript
// server/models/botConfig.js
const mongoose = require("mongoose");

const BotConfigSchema = new mongoose.Schema({
  guildId: {
    type: String,
    required: true,
    unique: true,
  },
  prefix: {
    type: String,
    default: "!",
  },
  welcomeMessage: {
    enabled: {
      type: Boolean,
      default: false,
    },
    message: {
      type: String,
      default: "Welcome to the server, {user}!",
    },
    channelId: {
      type: String,
      default: null,
    },
  },
  leaveMessage: {
    enabled: {
      type: Boolean,
      default: false,
    },
    message: {
      type: String,
      default: "Goodbye, {user}!",
    },
    channelId: {
      type: String,
      default: null,
    },
  },
  autoRole: {
    enabled: {
      type: Boolean,
      default: false,
    },
    roleId: {
      type: String,
      default: null,
    },
  },
  moderation: {
    enabled: {
      type: Boolean,
      default: false,
    },
    logChannelId: {
      type: String,
      default: null,
    },
    autoModeration: {
      enabled: {
        type: Boolean,
        default: false,
      },
      filterInvites: {
        type: Boolean,
        default: false,
      },
      filterLinks: {
        type: Boolean,
        default: false,
      },
      filterProfanity: {
        type: Boolean,
        default: false,
      },
    },
  },
  levelSystem: {
    enabled: {
      type: Boolean,
      default: false,
    },
    announcement: {
      type: Boolean,
      default: true,
    },
    announceChannelId: {
      type: String,
      default: null,
    },
    xpPerMessage: {
      type: Number,
      default: 10,
    },
    xpCooldown: {
      type: Number,
      default: 60, // Secondes
    },
  },
  customCommands: {
    type: Map,
    of: {
      response: String,
      createdBy: String,
      createdAt: {
        type: Date,
        default: Date.now,
      },
    },
    default: {},
  },
  disabledCommands: {
    type: [String],
    default: [],
  },
  updatedAt: {
    type: Date,
    default: Date.now,
  },
});

// Middleware pre-save pour mettre à jour le champ updatedAt
BotConfigSchema.pre("save", function (next) {
  this.updatedAt = new Date();
  next();
});

module.exports = mongoose.model("BotConfig", BotConfigSchema);
```

### Routes pour contrôler le bot directement

```javascript
// server/routes/bot.js
const express = require("express");
const router = express.Router();
const logger = require("../../src/utils/logger");

// Middleware pour vérifier l'authentification
const isAuthenticated = (req, res, next) => {
  if (req.isAuthenticated()) {
    return next();
  }
  res.status(401).json({ error: "Not authenticated" });
};

// Middleware pour vérifier l'administrateur du bot
const isBotAdmin = (req, res, next) => {
  if (req.user.isAdmin) {
    return next();
  }
  res.status(403).json({ error: "Insufficient permissions" });
};

// Obtenir le statut du bot
router.get("/status", isAuthenticated, async (req, res) => {
  try {
    // Vous devrez implémenter un moyen de communiquer avec votre bot
    // Par exemple via une API interne

    const status = {
      online: true,
      uptime: "5d 7h 23m",
      lastRestart: new Date(Date.now() - 5 * 24 * 60 * 60 * 1000), // 5 jours
      version: "1.0.0",
      shardCount: 1,
      memoryUsage: 156, // MB
      cpuUsage: 2.5, // %
      commandsProcessed: 5243,
    };

    res.json(status);
  } catch (error) {
    logger.error("Error fetching bot status:", error);
    res.status(500).json({ error: "Failed to fetch bot status" });
  }
});

// Redémarrer le bot (admin uniquement)
router.post("/restart", isAuthenticated, isBotAdmin, async (req, res) => {
  try {
    // Implémenter la logique pour redémarrer le bot
    // Par exemple, envoyer un signal à votre système de gestion de processus

    logger.info(
      `Bot restart requested by ${req.user.username}#${req.user.discriminator}`
    );

    // Simuler un délai pour le redémarrage
    setTimeout(() => {
      // Logique de redémarrage...
      logger.info("Bot restarted successfully");
    }, 2000);

    res.json({ success: true, message: "Bot restart initiated" });
  } catch (error) {
    logger.error("Error restarting bot:", error);
    res.status(500).json({ error: "Failed to restart bot" });
  }
});

// Mettre à jour le bot (admin uniquement)
router.post("/update", isAuthenticated, isBotAdmin, async (req, res) => {
  try {
    // Implémenter la logique pour mettre à jour le bot
    // Par exemple, pull des dernières modifications Git et redémarrage

    logger.info(
      `Bot update requested by ${req.user.username}#${req.user.discriminator}`
    );

    // Simuler une mise à jour
    setTimeout(() => {
      // Logique de mise à jour...
      logger.info("Bot updated successfully");
    }, 5000);

    res.json({ success: true, message: "Bot update initiated" });
  } catch (error) {
    logger.error("Error updating bot:", error);
    res.status(500).json({ error: "Failed to update bot" });
  }
});

// Obtenir les logs du bot (admin uniquement)
router.get("/logs", isAuthenticated, isBotAdmin, async (req, res) => {
  try {
    // Implémenter la logique pour récupérer les logs du bot

    const logs = [
      {
        level: "info",
        timestamp: new Date(Date.now() - 60000),
        message: "Bot started successfully",
      },
      {
        level: "info",
        timestamp: new Date(Date.now() - 50000),
        message: "Connected to Discord API",
      },
      {
        level: "info",
        timestamp: new Date(Date.now() - 40000),
        message: "Loaded 25 commands",
      },
      {
        level: "warn",
        timestamp: new Date(Date.now() - 30000),
        message: "Rate limit reached for endpoint /users",
      },
      {
        level: "error",
        timestamp: new Date(Date.now() - 20000),
        message: "Failed to process command: !stats",
      },
    ];

    res.json(logs);
  } catch (error) {
    logger.error("Error fetching bot logs:", error);
    res.status(500).json({ error: "Failed to fetch bot logs" });
  }
});

module.exports = router;
```

## Configuration du client React

Maintenant que nous avons configuré le backend, passons au frontend React :

### Installation des dépendances

```bash
npx create-react-app client
cd client
npm install axios react-router-dom react-bootstrap bootstrap chart.js react-chartjs-2 formik yup react-toastify
```

### Structure de base de l'application

```jsx
// client/src/App.js
import React, { useState, useEffect } from "react";
import {
  BrowserRouter as Router,
  Route,
  Routes,
  Navigate,
} from "react-router-dom";
import axios from "axios";
import { ToastContainer, toast } from "react-toastify";
import "react-toastify/dist/ReactToastify.css";
import "bootstrap/dist/css/bootstrap.min.css";

// Importation des composants
import Navbar from "./components/Navbar";
import Home from "./pages/Home";
import Login from "./pages/Login";
import Dashboard from "./pages/Dashboard";
import ServerSettings from "./pages/ServerSettings";
import AdminPanel from "./pages/AdminPanel";
import NotFound from "./pages/NotFound";

// Importation du service d'authentification
import AuthService from "./services/AuthService";

// Configuration d'Axios
axios.defaults.baseURL = process.env.REACT_APP_API_URL;
axios.defaults.withCredentials = true;

const App = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Vérifier l'état de l'authentification au chargement
    const checkAuth = async () => {
      try {
        const response = await AuthService.getStatus();
        if (response.authenticated) {
          setUser(response.user);
        }
      } catch (error) {
        console.error("Error checking authentication:", error);
        toast.error("Failed to verify authentication status");
      } finally {
        setLoading(false);
      }
    };

    checkAuth();
  }, []);

  // Composant de route protégée
  const PrivateRoute = ({ children }) => {
    if (loading) return <div>Loading...</div>;
    return user ? children : <Navigate to="/login" />;
  };

  // Composant de route admin
  const AdminRoute = ({ children }) => {
    if (loading) return <div>Loading...</div>;
    return user && user.isAdmin ? children : <Navigate to="/dashboard" />;
  };

  return (
    <Router>
      <ToastContainer position="top-right" autoClose={5000} />
      <Navbar user={user} setUser={setUser} />
      <div className="container mt-4">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/login" element={<Login />} />
          <Route
            path="/dashboard"
            element={
              <PrivateRoute>
                <Dashboard user={user} />
              </PrivateRoute>
            }
          />
          <Route
            path="/servers/:serverId"
            element={
              <PrivateRoute>
                <ServerSettings user={user} />
              </PrivateRoute>
            }
          />
          <Route
            path="/admin"
            element={
              <AdminRoute>
                <AdminPanel />
              </AdminRoute>
            }
          />
          <Route path="*" element={<NotFound />} />
        </Routes>
      </div>
    </Router>
  );
};

export default App;
```

### Service d'authentification

```jsx
// client/src/services/AuthService.js
import axios from "axios";

const AuthService = {
  /**
   * Obtenir l'état de l'authentification
   */
  getStatus: async () => {
    const response = await axios.get("/auth/status");
    return response.data;
  },

  /**
   * Obtenir la liste des serveurs de l'utilisateur
   */
  getGuilds: async () => {
    const response = await axios.get("/auth/guilds");
    return response.data;
  },

  /**
   * Déconnexion
   */
  logout: async () => {
    const response = await axios.get("/auth/logout");
    return response.data;
  },

  /**
   * URL de redirection pour l'authentification Discord
   */
  getDiscordAuthUrl: () => {
    return `${process.env.REACT_APP_API_URL}/auth/discord`;
  },
};

export default AuthService;
```

### Service API pour les statistiques et la configuration du bot

```jsx
// client/src/services/ApiService.js
import axios from "axios";

const ApiService = {
  /**
   * Obtenir les statistiques générales du bot
   */
  getStats: async () => {
    const response = await axios.get("/api/stats");
    return response.data;
  },

  /**
   * Obtenir la configuration d'un serveur
   */
  getGuildConfig: async (guildId) => {
    const response = await axios.get(`/api/guilds/${guildId}/config`);
    return response.data;
  },

  /**
   * Mettre à jour la configuration d'un serveur
   */
  updateGuildConfig: async (guildId, config) => {
    const response = await axios.put(`/api/guilds/${guildId}/config`, config);
    return response.data;
  },

  /**
   * Obtenir l'historique des commandes d'un serveur
   */
  getCommandHistory: async (guildId) => {
    const response = await axios.get(`/api/guilds/${guildId}/commands`);
    return response.data;
  },
};

export default ApiService;
```

### Service pour le contrôle direct du bot

```jsx
// client/src/services/BotService.js
import axios from "axios";

const BotService = {
  /**
   * Obtenir le statut du bot
   */
  getStatus: async () => {
    const response = await axios.get("/bot/status");
    return response.data;
  },

  /**
   * Redémarrer le bot
   */
  restart: async () => {
    const response = await axios.post("/bot/restart");
    return response.data;
  },

  /**
   * Mettre à jour le bot
   */
  update: async () => {
    const response = await axios.post("/bot/update");
    return response.data;
  },

  /**
   * Obtenir les logs du bot
   */
  getLogs: async () => {
    const response = await axios.get("/bot/logs");
    return response.data;
  },
};

export default BotService;
```

### Composant de barre de navigation

```jsx
// client/src/components/Navbar.js
import React from "react";
import { Link, useNavigate } from "react-router-dom";
import { Navbar, Nav, Container, Button } from "react-bootstrap";
import { toast } from "react-toastify";
import AuthService from "../services/AuthService";

const Navigation = ({ user, setUser }) => {
  const navigate = useNavigate();

  const handleLogout = async () => {
    try {
      await AuthService.logout();
      setUser(null);
      toast.success("Logged out successfully");
      navigate("/");
    } catch (error) {
      console.error("Error during logout:", error);
      toast.error("Failed to log out");
    }
  };

  return (
    <Navbar bg="dark" variant="dark" expand="lg">
      <Container>
        <Navbar.Brand as={Link} to="/">
          Bot Dashboard
        </Navbar.Brand>
        <Navbar.Toggle aria-controls="navbar-nav" />
        <Navbar.Collapse id="navbar-nav">
          <Nav className="me-auto">
            <Nav.Link as={Link} to="/">
              Home
            </Nav.Link>
            {user && (
              <Nav.Link as={Link} to="/dashboard">
                Dashboard
              </Nav.Link>
            )}
            {user && user.isAdmin && (
              <Nav.Link as={Link} to="/admin">
                Admin
              </Nav.Link>
            )}
          </Nav>
          <Nav>
            {user ? (
              <div className="d-flex align-items-center">
                <span className="text-light me-3">
                  Welcome, {user.username}
                </span>
                <Button variant="outline-light" onClick={handleLogout}>
                  Logout
                </Button>
              </div>
            ) : (
              <Button variant="outline-light" as={Link} to="/login">
                Login
              </Button>
            )}
          </Nav>
        </Navbar.Collapse>
      </Container>
    </Navbar>
  );
};

export default Navigation;
```

### Page d'accueil

```jsx
// client/src/pages/Home.js
import React from "react";
import { Link } from "react-router-dom";
import { Container, Row, Col, Card, Button } from "react-bootstrap";

const Home = () => {
  return (
    <Container>
      <Row className="my-5 text-center">
        <Col>
          <h1>Welcome to Bot Dashboard</h1>
          <p className="lead">
            Manage and configure your Discord bot with ease.
          </p>
        </Col>
      </Row>

      <Row className="my-5">
        <Col md={4} className="mb-4">
          <Card>
            <Card.Body>
              <Card.Title>Easy Configuration</Card.Title>
              <Card.Text>
                Configure your bot settings without having to use commands.
                Adjust welcome messages, auto-roles, moderation settings, and
                more.
              </Card.Text>
            </Card.Body>
          </Card>
        </Col>
        <Col md={4} className="mb-4">
          <Card>
            <Card.Body>
              <Card.Title>Real-time Statistics</Card.Title>
              <Card.Text>
                View detailed statistics about your bot's performance, including
                command usage, server activity, and more.
              </Card.Text>
            </Card.Body>
          </Card>
        </Col>
        <Col md={4} className="mb-4">
          <Card>
            <Card.Body>
              <Card.Title>Custom Commands</Card.Title>
              <Card.Text>
                Create and manage custom commands for your server, without
                writing a single line of code.
              </Card.Text>
            </Card.Body>
          </Card>
        </Col>
      </Row>

      <Row className="my-5 text-center">
        <Col>
          <p>Ready to get started?</p>
          <Button as={Link} to="/login" variant="primary" size="lg">
            Login with Discord
          </Button>
        </Col>
      </Row>
    </Container>
  );
};

export default Home;
```

### Page de connexion

```jsx
// client/src/pages/Login.js
import React from "react";
import { Container, Row, Col, Card, Button } from "react-bootstrap";
import AuthService from "../services/AuthService";

const Login = () => {
  const handleLogin = () => {
    window.location.href = AuthService.getDiscordAuthUrl();
  };

  return (
    <Container>
      <Row className="justify-content-center mt-5">
        <Col md={6}>
          <Card>
            <Card.Header as="h5" className="text-center">
              Login to Dashboard
            </Card.Header>
            <Card.Body className="text-center">
              <Card.Text>
                To access the dashboard, please login with your Discord account.
                This will allow you to manage your bot configurations.
              </Card.Text>
              <Button
                variant="primary"
                size="lg"
                onClick={handleLogin}
                className="mt-3"
              >
                <i className="fab fa-discord me-2"></i>
                Login with Discord
              </Button>
            </Card.Body>
          </Card>
        </Col>
      </Row>
    </Container>
  );
};

export default Login;
```

### Tableau de bord principal

```jsx
// client/src/pages/Dashboard.js
import React, { useState, useEffect } from "react";
import { Link } from "react-router-dom";
import { Container, Row, Col, Card, ListGroup, Spinner } from "react-bootstrap";
import { toast } from "react-toastify";
import AuthService from "../services/AuthService";
import ApiService from "../services/ApiService";

const Dashboard = ({ user }) => {
  const [guilds, setGuilds] = useState([]);
  const [stats, setStats] = useState(null);
  const [loading, setLoading] = useState(true);

  // Charger les données lors du montage du composant
  useEffect(() => {
    const fetchData = async () => {
      try {
        // Charger les serveurs et les statistiques en parallèle
        const [guildsResponse, statsResponse] = await Promise.all([
          AuthService.getGuilds(),
          ApiService.getStats(),
        ]);

        setGuilds(guildsResponse.guilds || []);
        setStats(statsResponse);
      } catch (error) {
        console.error("Error fetching dashboard data:", error);
        toast.error("Failed to load dashboard data");
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, []);

  // Filtrer les serveurs où l'utilisateur a des permissions d'administrateur
  const adminGuilds = guilds.filter(
    (guild) => guild.owner || (guild.permissions & 0x8) === 0x8
  );

  if (loading) {
    return (
      <Container className="text-center mt-5">
        <Spinner animation="border" role="status">
          <span className="visually-hidden">Loading...</span>
        </Spinner>
        <p className="mt-3">Loading dashboard data...</p>
      </Container>
    );
  }

  return (
    <Container>
      <h1 className="my-4">Dashboard</h1>

      {stats && (
        <Row className="mb-4">
          <Col md={3} className="mb-3">
            <Card className="text-center h-100">
              <Card.Body>
                <Card.Title>Servers</Card.Title>
                <h3>{stats.guilds}</h3>
              </Card.Body>
            </Card>
          </Col>
          <Col md={3} className="mb-3">
            <Card className="text-center h-100">
              <Card.Body>
                <Card.Title>Users</Card.Title>
                <h3>{stats.users}</h3>
              </Card.Body>
            </Card>
          </Col>
          <Col md={3} className="mb-3">
            <Card className="text-center h-100">
              <Card.Body>
                <Card.Title>Commands</Card.Title>
                <h3>{stats.commands}</h3>
              </Card.Body>
            </Card>
          </Col>
          <Col md={3} className="mb-3">
            <Card className="text-center h-100">
              <Card.Body>
                <Card.Title>Uptime</Card.Title>
                <h3>{stats.uptime}</h3>
              </Card.Body>
            </Card>
          </Col>
        </Row>
      )}

      <Row>
        <Col>
          <Card>
            <Card.Header>
              <h5 className="mb-0">Your Servers</h5>
            </Card.Header>
            <ListGroup variant="flush">
              {adminGuilds.length > 0 ? (
                adminGuilds.map((guild) => (
                  <ListGroup.Item
                    key={guild.id}
                    action
                    as={Link}
                    to={`/servers/${guild.id}`}
                    className="d-flex justify-content-between align-items-center"
                  >
                    <div className="d-flex align-items-center">
                      {guild.icon ? (
                        <img
                          src={`https://cdn.discordapp.com/icons/${guild.id}/${guild.icon}.png`}
                          alt={guild.name}
                          className="me-3 rounded-circle"
                          width="40"
                          height="40"
                        />
                      ) : (
                        <div
                          className="me-3 rounded-circle bg-secondary text-white d-flex align-items-center justify-content-center"
                          style={{ width: 40, height: 40 }}
                        >
                          {guild.name.charAt(0)}
                        </div>
                      )}
                      <span>{guild.name}</span>
                    </div>
                    <span className="text-muted">Configure</span>
                  </ListGroup.Item>
                ))
              ) : (
                <ListGroup.Item className="text-center py-4">
                  <p className="mb-0">
                    You don't have admin access to any servers with the bot.
                  </p>
                </ListGroup.Item>
              )}
            </ListGroup>
          </Card>
        </Col>
      </Row>
    </Container>
  );
};

export default Dashboard;
```

### Page de configuration du serveur

```jsx
// client/src/pages/ServerSettings.js
import React, { useState, useEffect } from "react";
import { useParams, useNavigate } from "react-router-dom";
import {
  Container,
  Row,
  Col,
  Card,
  Form,
  Button,
  Tab,
  Nav,
  Spinner,
  Alert,
} from "react-bootstrap";
import { toast } from "react-toastify";
import { Formik } from "formik";
import * as Yup from "yup";
import ApiService from "../services/ApiService";

// Schéma de validation pour le formulaire
const validationSchema = Yup.object().shape({
  prefix: Yup.string()
    .max(5, "Prefix must be 5 characters or less")
    .required("Prefix is required"),
  welcomeMessage: Yup.object().shape({
    enabled: Yup.boolean(),
    message: Yup.string().when("enabled", {
      is: true,
      then: Yup.string().required("Welcome message is required when enabled"),
    }),
    channelId: Yup.string().when("enabled", {
      is: true,
      then: Yup.string().required(
        "Channel is required when welcome message is enabled"
      ),
    }),
  }),
  autoRole: Yup.object().shape({
    enabled: Yup.boolean(),
    roleId: Yup.string().when("enabled", {
      is: true,
      then: Yup.string().required("Role is required when auto-role is enabled"),
    }),
  }),
});

const ServerSettings = () => {
  const { serverId } = useParams();
  const navigate = useNavigate();
  const [serverConfig, setServerConfig] = useState(null);
  const [loading, setLoading] = useState(true);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState(null);

  // Charger la configuration du serveur
  useEffect(() => {
    const fetchConfig = async () => {
      try {
        setLoading(true);
        const config = await ApiService.getGuildConfig(serverId);
        setServerConfig(config);
        setError(null);
      } catch (error) {
        console.error("Error fetching server config:", error);
        setError("Failed to load server configuration. Please try again.");
        toast.error("Failed to load server configuration");
      } finally {
        setLoading(false);
      }
    };

    fetchConfig();
  }, [serverId]);

  // Gestionnaire de soumission du formulaire
  const handleSubmit = async (values) => {
    try {
      setSaving(true);
      await ApiService.updateGuildConfig(serverId, values);
      setServerConfig(values);
      toast.success("Server configuration updated successfully");
    } catch (error) {
      console.error("Error updating server config:", error);
      toast.error("Failed to update server configuration");
    } finally {
      setSaving(false);
    }
  };

  if (loading) {
    return (
      <Container className="text-center mt-5">
        <Spinner animation="border" role="status">
          <span className="visually-hidden">
            Loading server configuration...
          </span>
        </Spinner>
        <p className="mt-3">Loading server configuration...</p>
      </Container>
    );
  }

  if (error) {
    return (
      <Container className="mt-5">
        <Alert variant="danger">
          <Alert.Heading>Error loading server configuration</Alert.Heading>
          <p>{error}</p>
          <hr />
          <div className="d-flex justify-content-end">
            <Button
              variant="outline-danger"
              onClick={() => navigate("/dashboard")}
            >
              Back to Dashboard
            </Button>
          </div>
        </Alert>
      </Container>
    );
  }

  return (
    <Container>
      <h1 className="my-4">Server Settings</h1>

      <Formik
        initialValues={serverConfig}
        validationSchema={validationSchema}
        onSubmit={handleSubmit}
        enableReinitialize={true}
      >
        {({
          values,
          errors,
          touched,
          handleChange,
          handleBlur,
          handleSubmit,
          setFieldValue,
        }) => (
          <Form onSubmit={handleSubmit}>
            <Tab.Container defaultActiveKey="general">
              <Row>
                <Col md={3} lg={2} className="mb-4">
                  <Nav variant="pills" className="flex-column">
                    <Nav.Item>
                      <Nav.Link eventKey="general">General</Nav.Link>
                    </Nav.Item>
                    <Nav.Item>
                      <Nav.Link eventKey="welcome">Welcome</Nav.Link>
                    </Nav.Item>
                    <Nav.Item>
                      <Nav.Link eventKey="moderation">Moderation</Nav.Link>
                    </Nav.Item>
                    <Nav.Item>
                      <Nav.Link eventKey="levels">Levels</Nav.Link>
                    </Nav.Item>
                    <Nav.Item>
                      <Nav.Link eventKey="commands">Commands</Nav.Link>
                    </Nav.Item>
                  </Nav>
                </Col>

                <Col md={9} lg={10}>
                  <Card>
                    <Card.Body>
                      <Tab.Content>
                        {/* Onglet Général */}
                        <Tab.Pane eventKey="general">
                          <h4 className="mb-3">General Settings</h4>
                          <Form.Group className="mb-3">
                            <Form.Label>Command Prefix</Form.Label>
                            <Form.Control
                              type="text"
                              name="prefix"
                              value={values.prefix}
                              onChange={handleChange}
                              onBlur={handleBlur}
                              isInvalid={touched.prefix && errors.prefix}
                            />
                            <Form.Control.Feedback type="invalid">
                              {errors.prefix}
                            </Form.Control.Feedback>
                            <Form.Text className="text-muted">
                              The prefix used to trigger bot commands (e.g., !,
                              $, /)
                            </Form.Text>
                          </Form.Group>
                        </Tab.Pane>

                        {/* Onglet Welcome */}
                        <Tab.Pane eventKey="welcome">
                          <h4 className="mb-3">Welcome Settings</h4>

                          <Form.Group className="mb-3">
                            <Form.Check
                              type="switch"
                              id="welcome-enabled"
                              label="Enable welcome messages"
                              name="welcomeMessage.enabled"
                              checked={values.welcomeMessage.enabled}
                              onChange={handleChange}
                            />
                          </Form.Group>

                          {values.welcomeMessage.enabled && (
                            <>
                              <Form.Group className="mb-3">
                                <Form.Label>Welcome Message</Form.Label>
                                <Form.Control
                                  as="textarea"
                                  rows={3}
                                  name="welcomeMessage.message"
                                  value={values.welcomeMessage.message}
                                  onChange={handleChange}
                                  onBlur={handleBlur}
                                  isInvalid={
                                    touched.welcomeMessage?.message &&
                                    errors.welcomeMessage?.message
                                  }
                                />
                                <Form.Control.Feedback type="invalid">
                                  {errors.welcomeMessage?.message}
                                </Form.Control.Feedback>
                                <Form.Text className="text-muted">
                                  Use {"{user}"} to mention the user,{" "}
                                  {"{server}"} for server name
                                </Form.Text>
                              </Form.Group>

                              <Form.Group className="mb-3">
                                <Form.Label>Channel</Form.Label>
                                <Form.Select
                                  name="welcomeMessage.channelId"
                                  value={values.welcomeMessage.channelId || ""}
                                  onChange={handleChange}
                                  onBlur={handleBlur}
                                  isInvalid={
                                    touched.welcomeMessage?.channelId &&
                                    errors.welcomeMessage?.channelId
                                  }
                                >
                                  <option value="">Select channel</option>
                                  <option value="channel1">General</option>
                                  <option value="channel2">Welcome</option>
                                  <option value="channel3">
                                    Introductions
                                  </option>
                                </Form.Select>
                                <Form.Control.Feedback type="invalid">
                                  {errors.welcomeMessage?.channelId}
                                </Form.Control.Feedback>
                              </Form.Group>
                            </>
                          )}

                          <h5 className="mt-4">Auto Role</h5>
                          <Form.Group className="mb-3">
                            <Form.Check
                              type="switch"
                              id="autorole-enabled"
                              label="Automatically assign role to new members"
                              name="autoRole.enabled"
                              checked={values.autoRole.enabled}
                              onChange={handleChange}
                            />
                          </Form.Group>

                          {values.autoRole.enabled && (
                            <Form.Group className="mb-3">
                              <Form.Label>Role</Form.Label>
                              <Form.Select
                                name="autoRole.roleId"
                                value={values.autoRole.roleId || ""}
                                onChange={handleChange}
                                onBlur={handleBlur}
                                isInvalid={
                                  touched.autoRole?.roleId &&
                                  errors.autoRole?.roleId
                                }
                              >
                                <option value="">Select role</option>
                                <option value="role1">Member</option>
                                <option value="role2">Newcomer</option>
                                <option value="role3">Guest</option>
                              </Form.Select>
                              <Form.Control.Feedback type="invalid">
                                {errors.autoRole?.roleId}
                              </Form.Control.Feedback>
                            </Form.Group>
                          )}
                        </Tab.Pane>

                        {/* Autres onglets à implémenter de manière similaire */}
                        <Tab.Pane eventKey="moderation">
                          <h4 className="mb-3">Moderation Settings</h4>
                          <Alert variant="info">
                            Moderation settings would be configured here.
                          </Alert>
                        </Tab.Pane>

                        <Tab.Pane eventKey="levels">
                          <h4 className="mb-3">Level System Settings</h4>
                          <Alert variant="info">
                            Level system settings would be configured here.
                          </Alert>
                        </Tab.Pane>

                        <Tab.Pane eventKey="commands">
                          <h4 className="mb-3">Custom Commands</h4>
                          <Alert variant="info">
                            Custom command settings would be configured here.
                          </Alert>
                        </Tab.Pane>
                      </Tab.Content>
                    </Card.Body>
                    <Card.Footer>
                      <div className="d-flex justify-content-between">
                        <Button
                          variant="secondary"
                          onClick={() => navigate("/dashboard")}
                        >
                          Back
                        </Button>
                        <Button
                          variant="success"
                          type="submit"
                          disabled={saving}
                        >
                          {saving ? (
                            <>
                              <Spinner
                                as="span"
                                animation="border"
                                size="sm"
                                role="status"
                                aria-hidden="true"
                                className="me-2"
                              />
                              Saving...
                            </>
                          ) : (
                            "Save Changes"
                          )}
                        </Button>
                      </div>
                    </Card.Footer>
                  </Card>
                </Col>
              </Row>
            </Tab.Container>
          </Form>
        )}
      </Formik>
    </Container>
  );
};

export default ServerSettings;
```

### Panneau d'administration

```jsx
// client/src/pages/AdminPanel.js
import React, { useState, useEffect } from "react";
import {
  Container,
  Row,
  Col,
  Card,
  ListGroup,
  Button,
  Spinner,
  Alert,
  Modal,
} from "react-bootstrap";
import { toast } from "react-toastify";
import { Line } from "react-chartjs-2";
import BotService from "../services/BotService";

const AdminPanel = () => {
  const [botStatus, setBotStatus] = useState(null);
  const [logs, setLogs] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [showRestartModal, setShowRestartModal] = useState(false);
  const [showUpdateModal, setShowUpdateModal] = useState(false);
  const [restarting, setRestarting] = useState(false);
  const [updating, setUpdating] = useState(false);

  // Charger les données du bot
  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);

        // Charger le statut et les logs en parallèle
        const [statusData, logsData] = await Promise.all([
          BotService.getStatus(),
          BotService.getLogs(),
        ]);

        setBotStatus(statusData);
        setLogs(logsData);
        setError(null);
      } catch (error) {
        console.error("Error fetching admin data:", error);
        setError("Failed to load bot information.");
        toast.error("Failed to load bot information");
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, []);

  // Gestion du redémarrage du bot
  const handleRestart = async () => {
    try {
      setRestarting(true);
      await BotService.restart();
      toast.success("Bot restart initiated");
      setShowRestartModal(false);
    } catch (error) {
      console.error("Error restarting bot:", error);
      toast.error("Failed to restart bot");
    } finally {
      setRestarting(false);
    }
  };

  // Gestion de la mise à jour du bot
  const handleUpdate = async () => {
    try {
      setUpdating(true);
      await BotService.update();
      toast.success("Bot update initiated");
      setShowUpdateModal(false);
    } catch (error) {
      console.error("Error updating bot:", error);
      toast.error("Failed to update bot");
    } finally {
      setUpdating(false);
    }
  };

  // Configuration du graphique
  const chartData = {
    labels: ["Day 1", "Day 2", "Day 3", "Day 4", "Day 5", "Day 6", "Day 7"],
    datasets: [
      {
        label: "Commands",
        data: [150, 230, 180, 340, 290, 270, 320],
        fill: false,
        backgroundColor: "rgb(75, 192, 192)",
        borderColor: "rgba(75, 192, 192, 0.2)",
      },
    ],
  };

  if (loading) {
    return (
      <Container className="text-center mt-5">
        <Spinner animation="border" role="status">
          <span className="visually-hidden">Loading admin panel...</span>
        </Spinner>
        <p className="mt-3">Loading admin panel...</p>
      </Container>
    );
  }

  if (error) {
    return (
      <Container className="mt-5">
        <Alert variant="danger">
          <Alert.Heading>Error loading admin panel</Alert.Heading>
          <p>{error}</p>
        </Alert>
      </Container>
    );
  }

  return (
    <Container>
      <h1 className="my-4">Admin Panel</h1>

      <Row className="mb-4">
        <Col>
          <Card>
            <Card.Header>
              <h5 className="mb-0">Bot Status</h5>
            </Card.Header>
            <Card.Body>
              <Row>
                <Col md={3} className="mb-3">
                  <Card className="text-center h-100">
                    <Card.Body>
                      <Card.Title>Status</Card.Title>
                      <h3
                        className={
                          botStatus.online ? "text-success" : "text-danger"
                        }
                      >
                        {botStatus.online ? "Online" : "Offline"}
                      </h3>
                    </Card.Body>
                  </Card>
                </Col>
                <Col md={3} className="mb-3">
                  <Card className="text-center h-100">
                    <Card.Body>
                      <Card.Title>Uptime</Card.Title>
                      <h3>{botStatus.uptime}</h3>
                    </Card.Body>
                  </Card>
                </Col>
                <Col md={3} className="mb-3">
                  <Card className="text-center h-100">
                    <Card.Body>
                      <Card.Title>Memory Usage</Card.Title>
                      <h3>{botStatus.memoryUsage} MB</h3>
                    </Card.Body>
                  </Card>
                </Col>
                <Col md={3} className="mb-3">
                  <Card className="text-center h-100">
                    <Card.Body>
                      <Card.Title>CPU Usage</Card.Title>
                      <h3>{botStatus.cpuUsage}%</h3>
                    </Card.Body>
                  </Card>
                </Col>
              </Row>

              <Row className="mt-3">
                <Col className="d-flex justify-content-end">
                  <Button
                    variant="warning"
                    className="me-2"
                    onClick={() => setShowRestartModal(true)}
                  >
                    Restart Bot
                  </Button>
                  <Button
                    variant="info"
                    onClick={() => setShowUpdateModal(true)}
                  >
                    Update Bot
                  </Button>
                </Col>
              </Row>
            </Card.Body>
          </Card>
        </Col>
      </Row>

      <Row className="mb-4">
        <Col>
          <Card>
            <Card.Header>
              <h5 className="mb-0">Command Usage (Last 7 Days)</h5>
            </Card.Header>
            <Card.Body>
              <Line data={chartData} />
            </Card.Body>
          </Card>
        </Col>
      </Row>

      <Row>
        <Col>
          <Card>
            <Card.Header>
              <h5 className="mb-0">Recent Logs</h5>
            </Card.Header>
            <ListGroup variant="flush">
              {logs.map((log, index) => (
                <ListGroup.Item
                  key={index}
                  className={`d-flex align-items-center ${
                    log.level === "error"
                      ? "text-danger"
                      : log.level === "warn"
                      ? "text-warning"
                      : ""
                  }`}
                >
                  <small className="text-muted me-3">
                    {new Date(log.timestamp).toLocaleString()}
                  </small>
                  <span className="me-2">
                    {log.level === "info" && (
                      <i className="fas fa-info-circle text-info"></i>
                    )}
                    {log.level === "warn" && (
                      <i className="fas fa-exclamation-triangle text-warning"></i>
                    )}
                    {log.level === "error" && (
                      <i className="fas fa-times-circle text-danger"></i>
                    )}
                  </span>
                  <span>{log.message}</span>
                </ListGroup.Item>
              ))}
            </ListGroup>
          </Card>
        </Col>
      </Row>

      {/* Modal de confirmation pour le redémarrage */}
      <Modal show={showRestartModal} onHide={() => setShowRestartModal(false)}>
        <Modal.Header closeButton>
          <Modal.Title>Confirm Restart</Modal.Title>
        </Modal.Header>
        <Modal.Body>
          Are you sure you want to restart the bot? This will temporarily
          disconnect the bot from all servers.
        </Modal.Body>
        <Modal.Footer>
          <Button
            variant="secondary"
            onClick={() => setShowRestartModal(false)}
          >
            Cancel
          </Button>
          <Button
            variant="warning"
            onClick={handleRestart}
            disabled={restarting}
          >
            {restarting ? (
              <>
                <Spinner
                  as="span"
                  animation="border"
                  size="sm"
                  role="status"
                  aria-hidden="true"
                  className="me-2"
                />
                Restarting...
              </>
            ) : (
              "Restart Bot"
            )}
          </Button>
        </Modal.Footer>
      </Modal>

      {/* Modal de confirmation pour la mise à jour */}
      <Modal show={showUpdateModal} onHide={() => setShowUpdateModal(false)}>
        <Modal.Header closeButton>
          <Modal.Title>Confirm Update</Modal.Title>
        </Modal.Header>
        <Modal.Body>
          Are you sure you want to update the bot? This will pull the latest
          changes and restart the bot.
        </Modal.Body>
        <Modal.Footer>
          <Button variant="secondary" onClick={() => setShowUpdateModal(false)}>
            Cancel
          </Button>
          <Button variant="info" onClick={handleUpdate} disabled={updating}>
            {updating ? (
              <>
                <Spinner
                  as="span"
                  animation="border"
                  size="sm"
                  role="status"
                  aria-hidden="true"
                  className="me-2"
                />
                Updating...
              </>
            ) : (
              "Update Bot"
            )}
          </Button>
        </Modal.Footer>
      </Modal>
    </Container>
  );
};

export default AdminPanel;
```

## Intégration avec les bots

### Communication entre le bot Discord et le tableau de bord

Pour que votre bot Discord puisse communiquer avec le tableau de bord, vous devez mettre en place un système de communication. Voici quelques approches possibles :

#### 1. Communication via la base de données

La méthode la plus simple consiste à utiliser la base de données comme intermédiaire :

```javascript
// Dans le bot Discord, src/utils/configManager.js
const mongoose = require("mongoose");
const BotConfig = require("../models/botConfig");
const logger = require("./logger");

class ConfigManager {
  constructor() {
    this.cache = new Map();
    this.refreshInterval = 60000; // 1 minute
  }

  // Initialiser le gestionnaire de configuration
  init() {
    // Charger toutes les configurations au démarrage
    this.refreshAllConfigs();

    // Configurer l'actualisation périodique
    setInterval(() => this.refreshAllConfigs(), this.refreshInterval);
  }

  // Rafraîchir toutes les configurations
  async refreshAllConfigs() {
    try {
      const configs = await BotConfig.find({});

      for (const config of configs) {
        this.cache.set(config.guildId, config);
      }

      logger.debug(`Refreshed ${configs.length} guild configurations`);
    } catch (error) {
      logger.error("Error refreshing configurations:", error);
    }
  }

  // Obtenir la configuration d'un serveur spécifique
  async getConfig(guildId) {
    // Vérifier si la configuration est en cache
    if (this.cache.has(guildId)) {
      return this.cache.get(guildId);
    }

    try {
      // Sinon, la charger depuis la base de données
      let config = await BotConfig.findOne({ guildId });

      if (!config) {
        // Créer une configuration par défaut si elle n'existe pas
        config = new BotConfig({ guildId });
        await config.save();
      }

      // Ajouter au cache
      this.cache.set(guildId, config);

      return config;
    } catch (error) {
      logger.error(`Error getting config for guild ${guildId}:`, error);
      return null;
    }
  }

  // Mettre à jour la configuration d'un serveur
  async updateConfig(guildId, updates) {
    try {
      // Mettre à jour dans la base de données
      const config = await BotConfig.findOneAndUpdate(
        { guildId },
        { $set: updates },
        { new: true, upsert: true }
      );

      // Mettre à jour le cache
      this.cache.set(guildId, config);

      logger.info(`Updated configuration for guild ${guildId}`);
      return config;
    } catch (error) {
      logger.error(`Error updating config for guild ${guildId}:`, error);
      return null;
    }
  }
}

const configManager = new ConfigManager();
module.exports = configManager;
```

#### 2. Communication via une API WebSocket

Pour des mises à jour en temps réel, vous pouvez utiliser WebSocket :

```javascript
// Dans le serveur du dashboard, server/websocket.js
const WebSocket = require("ws");
const logger = require("../src/utils/logger");

class WebSocketServer {
  constructor(server) {
    this.wss = new WebSocket.Server({ server });
    this.clients = new Map();
    this.setupEvents();
  }

  setupEvents() {
    this.wss.on("connection", (ws, req) => {
      const ip = req.socket.remoteAddress;
      logger.info(`New WebSocket connection from ${ip}`);

      // Générer un ID client unique
      const clientId = Date.now().toString();
      this.clients.set(clientId, ws);

      // Envoyer un message initial
      ws.send(
        JSON.stringify({
          type: "connection",
          clientId,
          message: "Connected to bot dashboard",
        })
      );

      // Gérer les messages entrants
      ws.on("message", (message) => {
        try {
          const data = JSON.parse(message);
          this.handleMessage(clientId, data);
        } catch (error) {
          logger.error("Error handling WebSocket message:", error);
        }
      });

      // Gérer la déconnexion
      ws.on("close", () => {
        logger.info(`WebSocket connection closed for client ${clientId}`);
        this.clients.delete(clientId);
      });
    });
  }

  handleMessage(clientId, data) {
    logger.debug(`Received message from client ${clientId}:`, data);

    // Traiter les différents types de messages
    switch (data.type) {
      case "ping":
        this.sendToClient(clientId, { type: "pong" });
        break;

      case "config_update":
        // Diffuser la mise à jour à tous les clients concernés
        this.broadcast({
          type: "config_update",
          guildId: data.guildId,
          config: data.config,
        });
        break;

      default:
        logger.warn(`Unknown message type: ${data.type}`);
    }
  }

  // Envoyer un message à un client spécifique
  sendToClient(clientId, data) {
    const client = this.clients.get(clientId);

    if (client && client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(data));
    }
  }

  // Diffuser un message à tous les clients
  broadcast(data) {
    this.wss.clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify(data));
      }
    });
  }
}

module.exports = WebSocketServer;
```

## Sécurité du tableau de bord

La sécurité est primordiale pour un tableau de bord d'administration. Voici quelques pratiques à suivre :

### Middleware de vérification CSRF

```javascript
// server/middleware/csrf.js
const crypto = require("crypto");

// Middleware pour générer des tokens CSRF
const generateCsrfToken = (req, res, next) => {
  if (!req.session.csrfToken) {
    req.session.csrfToken = crypto.randomBytes(32).toString("hex");
  }

  res.locals.csrfToken = req.session.csrfToken;
  next();
};

// Middleware pour valider les tokens CSRF
const validateCsrfToken = (req, res, next) => {
  // Ignorer les méthodes GET, HEAD, OPTIONS
  if (["GET", "HEAD", "OPTIONS"].includes(req.method)) {
    return next();
  }

  const token = req.headers["x-csrf-token"] || req.body._csrf;

  if (!token || token !== req.session.csrfToken) {
    return res.status(403).json({ error: "CSRF token validation failed" });
  }

  next();
};

module.exports = {
  generateCsrfToken,
  validateCsrfToken,
};
```

### Limitation du taux de requêtes

```javascript
// server/middleware/rateLimit.js
const rateLimit = require("express-rate-limit");

// Limiteur général
const generalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limite chaque IP à 100 requêtes par fenêtre
  standardHeaders: true, // Retourner les headers standard rate limit
  legacyHeaders: false, // Désactiver les headers X-RateLimit-*
  message: { error: "Too many requests, please try again later" },
});

// Limiteur pour l'authentification
const authLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 heure
  max: 10, // Limite chaque IP à 10 requêtes par heure
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many login attempts, please try again later" },
});

// Limiteur pour les opérations d'administration
const adminLimiter = rateLimit({
  windowMs: 5 * 60 * 1000, // 5 minutes
  max: 20, // Limite chaque IP à 20 requêtes par 5 minutes
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: "Too many admin operations, please try again later" },
});

module.exports = {
  generalLimiter,
  authLimiter,
  adminLimiter,
};
```

### Validation des entrées avec Joi

```javascript
// server/middleware/validator.js
const Joi = require("joi");

// Schéma de validation pour la configuration du bot
const configSchema = Joi.object({
  prefix: Joi.string().max(5).required(),
  welcomeMessage: Joi.object({
    enabled: Joi.boolean().required(),
    message: Joi.string().when("enabled", {
      is: true,
      then: Joi.string().required(),
    }),
    channelId: Joi.string().when("enabled", {
      is: true,
      then: Joi.string().required(),
    }),
  }).required(),
  autoRole: Joi.object({
    enabled: Joi.boolean().required(),
    roleId: Joi.string().when("enabled", {
      is: true,
      then: Joi.string().required(),
    }),
  }).required(),
  moderation: Joi.object({
    enabled: Joi.boolean().required(),
    logChannelId: Joi.string().when("enabled", {
      is: true,
      then: Joi.string().required(),
    }),
    autoModeration: Joi.object({
      enabled: Joi.boolean().required(),
      filterInvites: Joi.boolean(),
      filterLinks: Joi.boolean(),
      filterProfanity: Joi.boolean(),
    }).required(),
  }).required(),
});

// Middleware de validation
const validateConfig = (req, res, next) => {
  const { error, value } = configSchema.validate(req.body, {
    abortEarly: false,
    stripUnknown: true,
  });

  if (error) {
    const details = error.details.map((detail) => ({
      field: detail.path.join("."),
      message: detail.message,
    }));

    return res.status(400).json({
      error: "Validation failed",
      details,
    });
  }

  // Remplacer le corps de la requête par les données validées
  req.body = value;
  next();
};

module.exports = {
  validateConfig,
};
```

## Conclusion

Dans ce chapitre, nous avons exploré la conception et le développement d'un tableau de bord web complet pour gérer vos bots Discord et Twitch. Nous avons couvert :

1. L'architecture du tableau de bord avec un backend Express et un frontend React
2. La configuration de l'authentification OAuth2 avec Discord
3. La création d'API RESTful pour gérer la configuration des bots
4. Le développement d'interfaces utilisateur intuitives pour la gestion des serveurs
5. La mise en place de fonctionnalités avancées comme le panneau d'administration
6. L'implémentation de mécanismes de sécurité pour protéger votre tableau de bord

Un tableau de bord bien conçu améliore considérablement l'expérience utilisateur de votre bot en permettant une configuration facile et intuitive, tout en offrant aux administrateurs des outils de surveillance et de gestion puissants.

## Ressources supplémentaires

- [React Documentation](https://reactjs.org/docs/getting-started.html)
- [Express.js Documentation](https://expressjs.com/)
- [Discord OAuth2 Guide](https://discord.com/developers/docs/topics/oauth2)
- [React Bootstrap Components](https://react-bootstrap.github.io/components/alerts/)
- [Chart.js Documentation](https://www.chartjs.org/docs/latest/)
- [Formik - Form Management](https://formik.org/docs/overview)
- [Yup - Schema Validation](https://github.com/jquense/yup)
- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
