# Chapitre 1 - Introduction à Node.js et JavaScript

## Qu'est-ce que Node.js ?

Node.js est un environnement d'exécution JavaScript côté serveur qui permet d'exécuter du code JavaScript en dehors d'un navigateur web. Créé en 2009 par Ryan Dahl, Node.js est construit sur le moteur JavaScript V8 de Google Chrome.

### Principales caractéristiques de Node.js :

- **Asynchrone et événementiel** : Idéal pour les applications en temps réel comme les bots
- **Non-bloquant** : Peut gérer plusieurs opérations simultanément
- **Rapide** : Exécution efficace du code JavaScript
- **Écosystème riche** : Accès à plus d'un million de packages via npm
- **Multi-plateforme** : Fonctionne sur Windows, macOS et Linux

## Pourquoi Node.js pour les bots Discord et Twitch ?

Node.js présente plusieurs avantages pour le développement de bots :

1. **Excellente gestion des événements en temps réel** - Parfait pour réagir aux messages et aux actions
2. **Grand écosystème de bibliothèques** - Discord.js, tmi.js et d'autres bibliothèques spécifiques aux bots
3. **Faible consommation de ressources** - Permet d'héberger des bots sur des serveurs peu coûteux
4. **Facilité d'apprentissage** - Si vous connaissez JavaScript, vous pouvez rapidement développer avec Node.js

## Installation de Node.js sur Windows 11

1. Visitez le site officiel : [https://nodejs.org/](https://nodejs.org/)
2. Téléchargez la version LTS (Long-Term Support) recommandée pour la plupart des utilisateurs
3. Exécutez l'installateur et suivez les instructions à l'écran
4. Ouvrez l'invite de commande (cmd) ou PowerShell et vérifiez l'installation :

```bash
node --version
npm --version
```

Si les commandes affichent les numéros de version, l'installation est réussie !

## Structure d'un projet Node.js de base

Un projet Node.js typique pour un bot inclut généralement :

```
mon-bot-discord/
│
├── node_modules/    # Répertoire des dépendances (généré automatiquement)
├── src/             # Code source du bot
│   ├── commands/    # Commandes du bot
│   ├── events/      # Gestionnaires d'événements
│   └── index.js     # Point d'entrée principal
│
├── .env             # Variables d'environnement (tokens, clés d'API)
├── .gitignore       # Fichiers à ignorer dans Git
├── package.json     # Métadonnées et dépendances du projet
└── README.md        # Documentation du projet
```

## Votre premier programme Node.js

Créons un simple programme "Hello World" pour vérifier que tout fonctionne :

1. Créez un nouveau dossier pour votre projet
2. À l'intérieur, créez un fichier nommé `hello.js`
3. Ajoutez le code suivant :

```javascript
// Votre premier programme Node.js
console.log("Hello, Node.js!");

// Affichage de l'heure actuelle
const now = new Date();
console.log(`Il est actuellement ${now.toLocaleTimeString()}`);

// Un exemple simple de fonction
function saluer(nom) {
  return `Bonjour, ${nom} ! Bienvenue dans le monde de Node.js.`;
}

console.log(saluer("développeur de bot"));
```

4. Exécutez le programme en ouvrant un terminal dans ce dossier et en tapant :

```bash
node hello.js
```

Vous devriez voir la sortie avec les messages de bienvenue !

## Initialisation d'un projet Node.js

Pour créer un nouveau projet Node.js :

1. Ouvrez un terminal dans votre dossier de projet
2. Exécutez la commande :

```bash
npm init
```

3. Suivez les instructions pour configurer votre projet (ou utilisez `npm init -y` pour accepter toutes les valeurs par défaut)
4. Un fichier `package.json` sera créé, qui servira de manifeste pour votre projet

## npm - Le gestionnaire de paquets de Node.js

npm (Node Package Manager) est l'outil qui vous permettra d'installer et de gérer les bibliothèques externes.

### Commandes npm essentielles :

```bash
# Installer une dépendance
npm install nomdupaquet

# Installer une dépendance de développement
npm install --save-dev nomdupaquet

# Installer discord.js (que nous utiliserons plus tard)
npm install discord.js

# Installer globalement un paquet
npm install -g nomdupaquet

# Exécuter un script défini dans package.json
npm run nomscript
```

## Exercice pratique

Pour mettre en pratique ce que vous avez appris :

1. Créez un nouveau dossier nommé "mon-premier-bot"
2. Initialisez un projet Node.js (`npm init -y`)
3. Créez un fichier `index.js` avec le contenu suivant :

```javascript
// Informations sur l'environnement Node.js
console.log("Informations sur Node.js :");
console.log(`Version de Node.js : ${process.version}`);
console.log(`Système d'exploitation : ${process.platform}`);
console.log(`Répertoire de travail : ${process.cwd()}`);

// Compte à rebours pour le lancement du bot
console.log("\nPréparation du lancement du bot...");
let countdown = 5;

const timer = setInterval(() => {
  console.log(`Lancement dans ${countdown} secondes...`);
  countdown--;

  if (countdown < 0) {
    clearInterval(timer);
    console.log("🚀 Bot prêt à être développé !");
  }
}, 1000);
```

4. Exécutez le programme avec `node index.js`

## Conclusion

Dans ce premier chapitre, vous avez appris :

- Ce qu'est Node.js et pourquoi il est idéal pour le développement de bots
- Comment installer Node.js sur Windows 11
- Comment créer un projet Node.js simple
- Comment utiliser npm pour gérer les dépendances

Dans le prochain chapitre, nous explorerons les fondamentaux de JavaScript nécessaires pour le développement avec Node.js.

## Ressources supplémentaires

- [Documentation officielle de Node.js](https://nodejs.org/docs/latest-v16.x/api/)
- [npm Documentation](https://docs.npmjs.com/)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
