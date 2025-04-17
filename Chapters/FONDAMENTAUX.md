# Chapitre 2 - Les fondamentaux de JavaScript pour Node.js

Dans ce chapitre, nous allons explorer les concepts fondamentaux de JavaScript qui sont essentiels pour développer des applications Node.js, en particulier des bots Discord et Twitch.

## Variables et types de données

JavaScript utilise plusieurs façons de déclarer des variables, chacune ayant ses particularités :

```javascript
// var - ancienne façon (à éviter dans le code moderne)
var ancienneVariable = "Valeur";

// let - pour les variables dont la valeur peut changer
let compteur = 0;
compteur = compteur + 1; // Valide

// const - pour les variables dont la valeur ne doit pas changer
const TOKEN_BOT = "mon-token-secret";
// TOKEN_BOT = "autre-valeur"; // Erreur !
```

### Types de données principaux

JavaScript possède plusieurs types de données fondamentaux :

```javascript
// String (chaîne de caractères)
const nomUtilisateur = "UserDiscord123";

// Number (nombre)
const niveau = 42;
const ratio = 3.14;

// Boolean (booléen)
const estConnecte = true;

// Array (tableau)
const commandes = ["aide", "stats", "jouer", "stop"];

// Object (objet)
const utilisateur = {
  id: "123456789",
  nom: "GameMaster",
  roles: ["admin", "modérateur"],
  dateInscription: new Date("2023-01-15"),
};

// null (valeur nulle intentionnelle)
const jeuActuel = null;

// undefined (valeur non définie)
let prochainJeu;
console.log(prochainJeu); // undefined
```

### Vérification de types

```javascript
// Utilisation de typeof pour vérifier le type
console.log(typeof "texte"); // "string"
console.log(typeof 42); // "number"
console.log(typeof true); // "boolean"
console.log(typeof { nom: "valeur" }); // "object"
console.log(typeof ["élément1"]); // "object" (attention, les tableaux sont des objets en JS)
console.log(Array.isArray(["élément1"])); // true (méthode pour vérifier si c'est un tableau)
```

## Opérateurs

### Opérateurs arithmétiques

```javascript
let a = 10;
let b = 3;

console.log(a + b); // 13 (addition)
console.log(a - b); // 7 (soustraction)
console.log(a * b); // 30 (multiplication)
console.log(a / b); // 3.3333... (division)
console.log(a % b); // 1 (modulo - reste de la division)
console.log(a ** b); // 1000 (exponentiation - a à la puissance b)

// Incrémentation et décrémentation
let compteur = 0;
compteur++; // équivalent à: compteur = compteur + 1
console.log(compteur); // 1
compteur--; // équivalent à: compteur = compteur - 1
console.log(compteur); // 0
```

### Opérateurs de comparaison

```javascript
console.log(5 == "5"); // true (égalité avec conversion de type)
console.log(5 === "5"); // false (égalité stricte, sans conversion)
console.log(5 != "5"); // false (inégalité avec conversion)
console.log(5 !== "5"); // true (inégalité stricte)
console.log(10 > 5); // true (supérieur à)
console.log(10 >= 10); // true (supérieur ou égal à)
console.log(10 < 5); // false (inférieur à)
console.log(10 <= 10); // true (inférieur ou égal à)
```

### Opérateurs logiques

```javascript
// AND logique (&&) - vrai seulement si les deux conditions sont vraies
console.log(true && true); // true
console.log(true && false); // false

// OR logique (||) - vrai si au moins une condition est vraie
console.log(true || false); // true
console.log(false || false); // false

// NOT logique (!) - inverse la valeur
console.log(!true); // false
console.log(!false); // true

// Exemple concret pour un bot
const utilisateur = {
  nom: "User123",
  estAdmin: false,
  estModerateur: true,
};

// Vérification des permissions
if (utilisateur.estAdmin || utilisateur.estModerateur) {
  console.log("Accès autorisé aux commandes de modération");
} else {
  console.log("Accès refusé");
}
```

## Structures conditionnelles

### Instruction if-else

```javascript
const message = "!aide";

if (message.startsWith("!")) {
  console.log("C'est une commande");

  // Instruction if imbriquée
  if (message === "!aide") {
    console.log("Affichage de l'aide");
  } else if (message === "!stats") {
    console.log("Affichage des statistiques");
  } else {
    console.log("Commande inconnue");
  }
} else {
  console.log("Ce n'est pas une commande");
}
```

### Switch case

```javascript
const commande = "!jouer";

switch (commande) {
  case "!aide":
    console.log("Voici les commandes disponibles...");
    break;
  case "!stats":
    console.log("Voici vos statistiques...");
    break;
  case "!jouer":
    console.log("Lancement du jeu...");
    break;
  default:
    console.log("Commande non reconnue");
}
```

### Opérateur ternaire

```javascript
// condition ? valeurSiVrai : valeurSiFaux
const estEnLigne = true;
const status = estEnLigne ? "En ligne" : "Hors ligne";
console.log(status); // "En ligne"

// Exemple pour un bot Discord
const utilisateur = {
  nom: "GameMaster",
  roles: ["admin", "modérateur"],
};

const couleurNom = utilisateur.roles.includes("admin")
  ? "#ff0000" // Rouge pour les admins
  : "#0000ff"; // Bleu pour les autres
console.log(`La couleur du nom est ${couleurNom}`);
```

## Boucles et itérations

### Boucle for

```javascript
// Boucle for classique
for (let i = 0; i < 5; i++) {
  console.log(`Itération ${i + 1}`);
}

// Parcourir un tableau avec for
const commandes = ["!aide", "!stats", "!jouer", "!quitter"];
for (let i = 0; i < commandes.length; i++) {
  console.log(`Commande ${i + 1}: ${commandes[i]}`);
}
```

### Boucle for...of

```javascript
// Idéal pour parcourir les éléments d'un tableau
const utilisateurs = ["User1", "User2", "User3"];
for (const utilisateur of utilisateurs) {
  console.log(`Bienvenue, ${utilisateur} !`);
}
```

### Boucle for...in

```javascript
// Pour parcourir les propriétés d'un objet
const configBot = {
  prefix: "!",
  nom: "AwesomeBot",
  version: "1.0.0",
  auteur: "Vous",
};

for (const propriete in configBot) {
  console.log(`${propriete}: ${configBot[propriete]}`);
}
```

### Boucle while

```javascript
// S'exécute tant que la condition est vraie
let compteur = 0;
while (compteur < 5) {
  console.log(`Compteur: ${compteur}`);
  compteur++;
}
```

### Boucle do...while

```javascript
// S'exécute au moins une fois, puis tant que la condition est vraie
let essai = 1;
do {
  console.log(`Essai n°${essai}`);
  essai++;
} while (essai <= 3);
```

## Fonctions

Les fonctions sont essentielles en JavaScript, en particulier pour organiser votre code de bot.

### Déclaration de fonction

```javascript
// Déclaration de fonction classique
function saluer(nom) {
  return `Bonjour, ${nom} !`;
}

console.log(saluer("DiscordUser")); // "Bonjour, DiscordUser !"

// Fonction avec paramètres par défaut
function configurerBot(nom = "DefaultBot", prefix = "!") {
  return {
    nom: nom,
    prefix: prefix,
    dateCreation: new Date(),
  };
}

const monBot = configurerBot("SuperBot", "$");
console.log(monBot);
```

### Expressions de fonction

```javascript
// Fonction assignée à une variable
const calculerXP = function (niveau, bonus = 0) {
  return niveau * 100 + bonus;
};

console.log(calculerXP(5, 50)); // 550
```

### Fonctions fléchées

```javascript
// Syntaxe plus concise, particulièrement utile pour les callbacks
const multiplier = (a, b) => a * b;
console.log(multiplier(4, 5)); // 20

// Avec un bloc de code
const verifierPermission = (utilisateur, role) => {
  if (!utilisateur || !utilisateur.roles) return false;
  return utilisateur.roles.includes(role);
};

const user = { roles: ["membre", "vip"] };
console.log(verifierPermission(user, "admin")); // false
console.log(verifierPermission(user, "vip")); // true
```

### Closures (fermetures)

```javascript
// Une fonction qui retourne une fonction, gardant accès à son contexte
function creerCompteur() {
  let compte = 0;

  return function () {
    compte++;
    return compte;
  };
}

const compteur = creerCompteur();
console.log(compteur()); // 1
console.log(compteur()); // 2
console.log(compteur()); // 3

// Exemple pour un bot
function creerGestionnaireCommandes(prefix) {
  const commandes = {};

  // Retourne un objet avec des méthodes
  return {
    ajouter: function (nom, callback) {
      commandes[nom] = callback;
    },
    executer: function (message) {
      if (!message.startsWith(prefix)) return false;

      const nomCommande = message.slice(prefix.length).split(" ")[0];
      if (commandes[nomCommande]) {
        commandes[nomCommande]();
        return true;
      }
      return false;
    },
  };
}

const gestionnaire = creerGestionnaireCommandes("!");
gestionnaire.ajouter("aide", () => console.log("Voici l'aide"));
gestionnaire.ajouter("stats", () => console.log("Voici les stats"));

gestionnaire.executer("!aide"); // Affiche: Voici l'aide
gestionnaire.executer("!stats"); // Affiche: Voici les stats
gestionnaire.executer("!inconnu"); // Ne fait rien, retourne false
```

## Tableaux et méthodes de tableau

Les tableaux sont fréquemment utilisés dans le développement de bots pour stocker des commandes, des utilisateurs, etc.

```javascript
// Création d'un tableau
const commandes = ["!aide", "!jouer", "!stats"];

// Accès aux éléments (l'indexation commence à 0)
console.log(commandes[0]); // "!aide"
console.log(commandes[2]); // "!stats"

// Modifier un élément
commandes[1] = "!game";
console.log(commandes); // ["!aide", "!game", "!stats"]

// Ajouter des éléments
commandes.push("!quitter"); // Ajoute à la fin
commandes.unshift("!start"); // Ajoute au début

// Supprimer des éléments
const dernier = commandes.pop(); // Supprime et retourne le dernier
const premier = commandes.shift(); // Supprime et retourne le premier

// Trouver l'index d'un élément
const index = commandes.indexOf("!stats");
console.log(index); // 1 (après nos modifications)

// Vérifier si un tableau contient un élément
const contientAide = commandes.includes("!aide");
console.log(contientAide); // true
```

### Méthodes avancées de tableau

```javascript
const utilisateurs = [
  { id: "1", nom: "Alice", niveau: 5, estAdmin: false },
  { id: "2", nom: "Bob", niveau: 3, estAdmin: false },
  { id: "3", nom: "Charlie", niveau: 8, estAdmin: true },
  { id: "4", nom: "David", niveau: 2, estAdmin: false },
  { id: "5", nom: "Eve", niveau: 7, estAdmin: true },
];

// forEach - exécuter une fonction pour chaque élément
utilisateurs.forEach((utilisateur) => {
  console.log(`${utilisateur.nom} est niveau ${utilisateur.niveau}`);
});

// map - transformer chaque élément et retourner un nouveau tableau
const noms = utilisateurs.map((utilisateur) => utilisateur.nom);
console.log(noms); // ["Alice", "Bob", "Charlie", "David", "Eve"]

// filter - créer un nouveau tableau avec les éléments qui satisfont une condition
const admins = utilisateurs.filter((utilisateur) => utilisateur.estAdmin);
console.log(admins); // [{ id: "3", nom: "Charlie", ... }, { id: "5", nom: "Eve", ... }]

// find - trouver le premier élément qui satisfait une condition
const niveauEleve = utilisateurs.find((utilisateur) => utilisateur.niveau > 5);
console.log(niveauEleve); // { id: "3", nom: "Charlie", ... }

// some - vérifie si au moins un élément satisfait une condition
const aAdmin = utilisateurs.some((utilisateur) => utilisateur.estAdmin);
console.log(aAdmin); // true

// every - vérifie si tous les éléments satisfont une condition
const tousAdmins = utilisateurs.every((utilisateur) => utilisateur.estAdmin);
console.log(tousAdmins); // false

// sort - trier le tableau
const utilisateursParNiveau = [...utilisateurs].sort(
  (a, b) => b.niveau - a.niveau
);
console.log(utilisateursParNiveau[0].nom); // "Charlie" (niveau 8)

// reduce - réduire le tableau à une seule valeur
const niveauTotal = utilisateurs.reduce(
  (total, utilisateur) => total + utilisateur.niveau,
  0
);
console.log(niveauTotal); // 25
```

## Objets et manipulation d'objets

Dans le développement de bots, vous utiliserez souvent des objets pour stocker des données structurées.

```javascript
// Création d'un objet
const bot = {
  nom: "SuperBot",
  version: "1.0.0",
  prefix: "!",
  commandes: ["aide", "stats", "jouer"],
  configuration: {
    couleur: "#ff0000",
    reponseAutomatique: true,
  },
  demarrer: function () {
    console.log(`${this.nom} v${this.version} démarre...`);
  },
  arreter: function () {
    console.log(`${this.nom} s'arrête...`);
  },
};

// Accès aux propriétés
console.log(bot.nom); // "SuperBot"
console.log(bot["version"]); // "1.0.0" (notation alternative)
console.log(bot.configuration.couleur); // "#ff0000"

// Appel de méthodes
bot.demarrer(); // "SuperBot v1.0.0 démarre..."

// Modifier des propriétés
bot.prefix = "$";
bot.configuration.reponseAutomatique = false;

// Ajouter des propriétés
bot.dateLancement = new Date();
bot.redemarrer = function () {
  this.arreter();
  this.demarrer();
};

// Vérifier si une propriété existe
console.log("nom" in bot); // true
console.log(bot.hasOwnProperty("auteur")); // false

// Obtenir toutes les clés
const proprietes = Object.keys(bot);
console.log(proprietes); // ["nom", "version", "prefix", ...]

// Obtenir toutes les valeurs
const valeurs = Object.values(bot);
console.log(valeurs); // ["SuperBot", "1.0.0", "!", ...]

// Obtenir les paires clé-valeur
const entrees = Object.entries(bot);
console.log(entrees); // [["nom", "SuperBot"], ["version", "1.0.0"], ...]
```

### Déstructuration d'objets

```javascript
// Extraire des propriétés spécifiques dans des variables
const { nom, version, prefix } = bot;
console.log(nom); // "SuperBot"
console.log(version); // "1.0.0"

// Déstructuration avec renommage
const { nom: nomBot, configuration: config } = bot;
console.log(nomBot); // "SuperBot"
console.log(config.couleur); // "#ff0000"

// Déstructuration avec valeurs par défaut
const { auteur = "Anonyme" } = bot;
console.log(auteur); // "Anonyme"

// Déstructuration dans les paramètres de fonction
function afficherInfoBot({ nom, version, prefix }) {
  console.log(`Bot: ${nom} v${version}, préfixe: ${prefix}`);
}

afficherInfoBot(bot); // "Bot: SuperBot v1.0.0, préfixe: $"
```

## Exercices pratiques

### Exercice 1 : Système de commandes simple

Créez un gestionnaire de commandes simple pour un bot :

```javascript
// gestionnaire-commandes.js
const prefixCommande = "!";
const commandes = {};

// Fonction pour enregistrer une commande
function enregistrerCommande(nom, description, callback) {
  commandes[nom] = {
    description: description,
    execute: callback,
  };
  console.log(`Commande ${prefixCommande}${nom} enregistrée`);
}

// Fonction pour exécuter une commande
function traiterMessage(message) {
  // Vérifier si le message commence par le préfixe
  if (!message.startsWith(prefixCommande)) return false;

  // Extraire le nom de la commande et les arguments
  const args = message.slice(prefixCommande.length).trim().split(/\s+/);
  const commandName = args.shift().toLowerCase();

  // Vérifier si la commande existe
  if (!commandes[commandName]) {
    console.log(`Commande inconnue: ${prefixCommande}${commandName}`);
    return false;
  }

  // Exécuter la commande
  try {
    commandes[commandName].execute(args);
    return true;
  } catch (error) {
    console.error(
      `Erreur lors de l'exécution de la commande ${commandName}:`,
      error
    );
    return false;
  }
}

// Fonction pour afficher l'aide
function afficherAide() {
  console.log("=== Liste des commandes disponibles ===");
  Object.keys(commandes).forEach((nom) => {
    console.log(`${prefixCommande}${nom} - ${commandes[nom].description}`);
  });
}

// Enregistrer quelques commandes
enregistrerCommande("aide", "Affiche la liste des commandes", () => {
  afficherAide();
});

enregistrerCommande("salut", "Salue l'utilisateur", (args) => {
  const nom = args.length > 0 ? args[0] : "utilisateur";
  console.log(`Bonjour, ${nom} ! Comment vas-tu aujourd'hui ?`);
});

enregistrerCommande("roll", "Lance un dé", (args) => {
  const max = args.length > 0 ? parseInt(args[0]) : 6;
  const resultat = Math.floor(Math.random() * max) + 1;
  console.log(`🎲 Vous avez obtenu ${resultat} (1-${max})`);
});

// Tester notre système de commandes
console.log("\n=== Test de notre système de commandes ===\n");
traiterMessage("!aide");
traiterMessage("!salut");
traiterMessage("!salut Alice");
traiterMessage("!roll");
traiterMessage("!roll 20");
traiterMessage("!inconnu");
traiterMessage("Ceci n'est pas une commande");
```

### Exercice 2 : Système de niveaux pour utilisateurs

Créez un système simple de gestion de niveaux pour les utilisateurs d'un serveur :

```javascript
// systeme-niveaux.js
const utilisateurs = [
  { id: "user1", nom: "Alice", xp: 230, dernierMessage: new Date() },
  { id: "user2", nom: "Bob", xp: 580, dernierMessage: new Date() },
  { id: "user3", nom: "Charlie", xp: 120, dernierMessage: new Date() },
];

// Fonction pour calculer le niveau en fonction de l'XP
function calculerNiveau(xp) {
  return Math.floor(Math.sqrt(xp) / 5) + 1;
}

// Fonction pour ajouter de l'XP à un utilisateur
function ajouterXP(userId, amount) {
  const utilisateur = utilisateurs.find((user) => user.id === userId);

  if (!utilisateur) {
    console.log(`Utilisateur ${userId} non trouvé`);
    return null;
  }

  // Vérifier si l'utilisateur peut gagner de l'XP (une fois par minute)
  const maintenant = new Date();
  const dernierMessage = new Date(utilisateur.dernierMessage);
  const diffMinutes = (maintenant - dernierMessage) / (1000 * 60);

  if (diffMinutes < 1) {
    console.log(`${utilisateur.nom} doit attendre avant de gagner plus d'XP`);
    return utilisateur;
  }

  // Calculer le niveau actuel avant l'ajout
  const niveauAvant = calculerNiveau(utilisateur.xp);

  // Ajouter de l'XP
  utilisateur.xp += amount;
  utilisateur.dernierMessage = maintenant;

  // Calculer le nouveau niveau
  const niveauApres = calculerNiveau(utilisateur.xp);

  // Vérifier si l'utilisateur a monté de niveau
  if (niveauApres > niveauAvant) {
    console.log(`🎉 ${utilisateur.nom} est passé au niveau ${niveauApres} !`);
  } else {
    console.log(
      `${utilisateur.nom} a gagné ${amount} XP (total: ${utilisateur.xp})`
    );
  }

  return utilisateur;
}

// Fonction pour afficher les statistiques d'un utilisateur
function afficherStats(userId) {
  const utilisateur = utilisateurs.find((user) => user.id === userId);

  if (!utilisateur) {
    console.log(`Utilisateur ${userId} non trouvé`);
    return;
  }

  const niveau = calculerNiveau(utilisateur.xp);
  const xpPourNiveauSuivant = Math.pow((niveau + 1 - 1) * 5, 2);
  const xpManquante = xpPourNiveauSuivant - utilisateur.xp;

  console.log(`=== Statistiques de ${utilisateur.nom} ===`);
  console.log(`Niveau: ${niveau}`);
  console.log(`XP: ${utilisateur.xp}/${xpPourNiveauSuivant}`);
  console.log(`XP manquante pour le niveau suivant: ${xpManquante}`);
  console.log(
    `Dernier message: ${utilisateur.dernierMessage.toLocaleString()}`
  );
}

// Fonction pour afficher le classement des utilisateurs
function afficherClassement() {
  console.log("=== Classement des utilisateurs ===");

  // Trier les utilisateurs par XP (décroissant)
  const classement = [...utilisateurs].sort((a, b) => b.xp - a.xp);

  classement.forEach((user, index) => {
    const niveau = calculerNiveau(user.xp);
    console.log(`${index + 1}. ${user.nom} - Niveau ${niveau} (${user.xp} XP)`);
  });
}

// Test du système
console.log("\n=== Test du système de niveaux ===\n");

// Afficher le classement initial
afficherClassement();

// Ajouter de l'XP à quelques utilisateurs
console.log("\n=== Ajout d'XP ===\n");
ajouterXP("user3", 500);
ajouterXP("user1", 300);

// Essayer d'ajouter de l'XP trop rapidement
ajouterXP("user3", 100);

// Permettons à user2 de gagner de l'XP en modifiant sa date de dernier message
utilisateurs.find((user) => user.id === "user2").dernierMessage = new Date(
  Date.now() - 2 * 60 * 1000
); // 2 minutes plus tôt
ajouterXP("user2", 200);

// Afficher les statistiques d'un utilisateur
console.log("\n=== Statistiques détaillées ===\n");
afficherStats("user3");

// Afficher le classement mis à jour
console.log("\n=== Classement mis à jour ===\n");
afficherClassement();
```

## Conclusion

Dans ce chapitre, vous avez appris les fondamentaux de JavaScript qui sont essentiels pour le développement avec Node.js :

- Variables et types de données
- Opérateurs et expressions
- Structures conditionnelles
- Boucles et itérations
- Fonctions et closures
- Manipulation de tableaux et d'objets

Ces concepts sont la base sur laquelle vous construirez vos bots Discord et Twitch. Dans le prochain chapitre, nous explorerons les concepts avancés de JavaScript et la gestion asynchrone, qui sont cruciaux pour le développement de bots réactifs et performants.

## Ressources supplémentaires

- [MDN Web Docs JavaScript](https://developer.mozilla.org/fr/docs/Web/JavaScript) - Documentation complète
- [JavaScript.info](https://fr.javascript.info/) - Tutoriels modernes
- [JavaScript Garden](https://bonsaiden.github.io/JavaScript-Garden/fr/) - Conseils sur les particularités de JavaScript
