# Chapitre 3 - Asynchrone et Promises en JavaScript

L'un des aspects les plus importants du développement de bots avec Node.js est la gestion des opérations asynchrones. Pour créer des bots Discord ou Twitch réactifs et efficaces, il est essentiel de comprendre ces concepts.

## Introduction à l'asynchrone en JavaScript

JavaScript est **mono-thread**, ce qui signifie qu'il ne peut exécuter qu'une seule instruction à la fois. Cependant, de nombreuses opérations dans le développement de bots sont potentiellement lentes :

- Envoi/réception de messages sur Discord/Twitch
- Requêtes vers des API externes
- Lectures/écritures de fichiers
- Opérations de base de données

Si ces opérations étaient exécutées de manière synchrone (bloquante), votre bot semblerait figé pendant leur exécution. C'est pourquoi Node.js utilise un modèle asynchrone non-bloquant.

## Callbacks : la méthode traditionnelle

Les callbacks sont la méthode historique pour gérer l'asynchrone en JavaScript.

```javascript
// Exemple avec setTimeout (simule une opération asynchrone)
console.log("Démarrage du bot...");

setTimeout(function () {
  console.log("Le bot est prêt !");
}, 2000); // Attendre 2 secondes

console.log("Chargement des commandes...");

// Résultat :
// "Démarrage du bot..."
// "Chargement des commandes..."
// "Le bot est prêt !" (après 2 secondes)
```

### Problème des callbacks imbriqués (callback hell)

```javascript
// Simulation d'opérations asynchrones avec des callbacks
function chargerConfiguration(callback) {
  setTimeout(() => {
    console.log("Configuration chargée");
    callback({ prefix: "!", name: "MonBot" });
  }, 1000);
}

function connecterAPI(config, callback) {
  setTimeout(() => {
    console.log(`Connexion à l'API avec ${config.name}`);
    callback("api-token-123");
  }, 1000);
}

function chargerCommandes(token, callback) {
  setTimeout(() => {
    console.log(`Chargement des commandes avec token: ${token}`);
    callback(["aide", "jouer", "stats"]);
  }, 1000);
}

// Utilisation - remarquez l'imbrication (callback hell)
console.log("Démarrage du bot...");
chargerConfiguration((config) => {
  connecterAPI(config, (token) => {
    chargerCommandes(token, (commandes) => {
      console.log(`Bot prêt avec ${commandes.length} commandes !`);
      // Imaginez plus de niveaux d'imbrication...
    });
  });
});
```

Ce code devient rapidement difficile à lire et à maintenir à mesure que la complexité augmente.

## Promises : une approche plus moderne

Les Promises sont un moyen élégant de gérer l'asynchrone en JavaScript. Une Promise représente une opération qui n'est pas encore terminée mais qui finira par être résolue (succès) ou rejetée (échec).

### Anatomie d'une Promise

```javascript
// Création d'une Promise
const maPromesse = new Promise((resolve, reject) => {
  // Opération asynchrone
  setTimeout(() => {
    const succes = true; // Simulons un succès

    if (succes) {
      resolve("Opération réussie !"); // Résolution de la promesse
    } else {
      reject(new Error("Échec de l'opération")); // Rejet de la promesse
    }
  }, 1000);
});

// Utilisation de la Promise
maPromesse
  .then((resultat) => {
    console.log("Succès:", resultat);
  })
  .catch((erreur) => {
    console.error("Erreur:", erreur.message);
  })
  .finally(() => {
    console.log("Opération terminée (succès ou échec)");
  });
```

### Conversion des callbacks en Promises

Réécrivons l'exemple précédent avec des Promises :

```javascript
// Conversion des fonctions callback en Promise
function chargerConfiguration() {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Configuration chargée");
      resolve({ prefix: "!", name: "MonBot" });
    }, 1000);
  });
}

function connecterAPI(config) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`Connexion à l'API avec ${config.name}`);
      resolve("api-token-123");
    }, 1000);
  });
}

function chargerCommandes(token) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`Chargement des commandes avec token: ${token}`);
      resolve(["aide", "jouer", "stats"]);
    }, 1000);
  });
}

// Utilisation - chaînage de Promises beaucoup plus lisible
console.log("Démarrage du bot...");
chargerConfiguration()
  .then((config) => connecterAPI(config))
  .then((token) => chargerCommandes(token))
  .then((commandes) => {
    console.log(`Bot prêt avec ${commandes.length} commandes !`);
  })
  .catch((erreur) => {
    console.error("Erreur lors du démarrage:", erreur);
  });
```

### Méthodes utiles pour les Promises

#### Promise.all - Attendre plusieurs promesses en parallèle

```javascript
// Chargement parallèle de plusieurs ressources
function chargerUtilisateurs() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(["user1", "user2", "user3"]);
    }, 1500);
  });
}

function chargerCanaux() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(["général", "jeux", "support"]);
    }, 1000);
  });
}

function chargerRoles() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(["admin", "modérateur", "membre"]);
    }, 2000);
  });
}

// Chargement parallèle
console.time("temps de chargement");
Promise.all([chargerUtilisateurs(), chargerCanaux(), chargerRoles()])
  .then(([utilisateurs, canaux, roles]) => {
    console.log("Utilisateurs:", utilisateurs);
    console.log("Canaux:", canaux);
    console.log("Rôles:", roles);
    console.timeEnd("temps de chargement"); // ~2 secondes (temps de la plus longue)
  })
  .catch((erreur) => {
    console.error("Erreur de chargement:", erreur);
  });
```

#### Promise.race - La première promesse qui se termine

```javascript
// Exemple d'une course entre deux sources de données
function rechercheLocale(terme) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Résultat local trouvé");
      resolve(`Résultat local pour "${terme}"`);
    }, 500); // Rapide mais peut-être moins complet
  });
}

function rechercheAPI(terme) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("Résultat API trouvé");
      resolve(`Résultat détaillé API pour "${terme}"`);
    }, 1000); // Plus lent mais plus complet
  });
}

// Utiliser le premier résultat disponible
const terme = "javascript";
Promise.race([rechercheLocale(terme), rechercheAPI(terme)])
  .then((resultat) => {
    console.log("Premier résultat:", resultat);
  })
  .catch((erreur) => {
    console.error("Erreur de recherche:", erreur);
  });
```

#### Promise.allSettled - Attendre toutes les promesses se terminent (réussies ou échouées)

```javascript
// Exemple de traitement multiple qui tolère les échecs partiels
function envoyerMessage(canal, message) {
  return new Promise((resolve, reject) => {
    const reussite = Math.random() > 0.3; // 70% de réussite

    setTimeout(() => {
      if (reussite) {
        resolve(`Message envoyé dans ${canal}: ${message}`);
      } else {
        reject(new Error(`Échec d'envoi dans ${canal}`));
      }
    }, 500);
  });
}

// Envoyer à plusieurs canaux et rapporter tous les résultats
const canaux = ["général", "annonces", "support", "feedback"];
const message = "Le bot vient d'être mis à jour !";

const promises = canaux.map((canal) => envoyerMessage(canal, message));

Promise.allSettled(promises).then((resultats) => {
  console.log("=== Rapport d'envoi ===");

  resultats.forEach((resultat, index) => {
    const canal = canaux[index];

    if (resultat.status === "fulfilled") {
      console.log(`✅ ${canal}: Message envoyé avec succès`);
    } else {
      console.log(`❌ ${canal}: Échec - ${resultat.reason.message}`);
    }
  });

  // Statistiques
  const reussites = resultats.filter((r) => r.status === "fulfilled").length;
  console.log(`\nRésumé: ${reussites}/${resultats.length} messages envoyés`);
});
```

## Async/Await : l'asynchrone qui ressemble à du synchrone

Async/await est une syntaxe qui simplifie encore plus le travail avec les Promises, rendant le code asynchrone plus facile à lire et à écrire.

```javascript
// Utilisation d'async/await avec les fonctions précédentes
async function demarrerBot() {
  try {
    console.log("Démarrage du bot...");

    // Chaque await suspend l'exécution jusqu'à la résolution de la promesse
    const config = await chargerConfiguration();
    const token = await connecterAPI(config);
    const commandes = await chargerCommandes(token);

    console.log(`Bot prêt avec ${commandes.length} commandes !`);
    return commandes;
  } catch (erreur) {
    console.error("Erreur lors du démarrage:", erreur);
    throw erreur; // Propager l'erreur si nécessaire
  }
}

// Appel de la fonction asynchrone
demarrerBot()
  .then((commandes) => {
    console.log("Commandes disponibles:", commandes);
  })
  .catch((erreur) => {
    console.error("Échec du démarrage:", erreur);
  });
```

### Traitements parallèles avec async/await

```javascript
// Combinaison de async/await avec Promise.all pour le parallélisme
async function initialiserServeur() {
  console.time("initialisation");
  try {
    // Exécution parallèle avec Promise.all
    const [utilisateurs, canaux, roles] = await Promise.all([
      chargerUtilisateurs(),
      chargerCanaux(),
      chargerRoles(),
    ]);

    console.log("=== Initialisation du serveur ===");
    console.log(`Utilisateurs chargés: ${utilisateurs.length}`);
    console.log(`Canaux chargés: ${canaux.length}`);
    console.log(`Rôles chargés: ${roles.length}`);

    console.timeEnd("initialisation");

    return {
      utilisateurs,
      canaux,
      roles,
      dateInitialisation: new Date(),
    };
  } catch (erreur) {
    console.error("Erreur d'initialisation:", erreur);
    throw erreur;
  }
}

// Appel de la fonction
initialiserServeur().then((etat) => {
  console.log("Serveur initialisé à:", etat.dateInitialisation);
});
```

### Boucles asynchrones avec async/await

```javascript
// Traitement séquentiel d'une liste d'éléments
async function envoyerMessagesSequentiels(canaux, message) {
  console.log(`Envoi du message à ${canaux.length} canaux séquentiellement...`);

  for (const canal of canaux) {
    try {
      // Attendre chaque envoi avant de passer au suivant
      await new Promise((resolve) => {
        setTimeout(() => {
          console.log(`Message envoyé à ${canal}: "${message}"`);
          resolve();
        }, 500);
      });
    } catch (erreur) {
      console.error(`Erreur lors de l'envoi à ${canal}:`, erreur);
      // Continuer avec le canal suivant malgré l'erreur
    }
  }

  console.log("Tous les messages ont été envoyés séquentiellement");
}

// Traitement parallèle d'une liste d'éléments
async function envoyerMessagesParalleles(canaux, message) {
  console.log(`Envoi du message à ${canaux.length} canaux en parallèle...`);

  const promesses = canaux.map((canal) => {
    return new Promise((resolve) => {
      setTimeout(() => {
        console.log(`Message envoyé à ${canal}: "${message}"`);
        resolve(canal);
      }, 500);
    });
  });

  // Attendre que tous les envois soient terminés
  const resultats = await Promise.all(promesses);
  console.log(
    `Tous les messages ont été envoyés en parallèle à: ${resultats.join(", ")}`
  );
}

// Démonstration
const canaux = ["général", "annonces", "support", "aide", "discussion"];
const message = "Test des envois de messages";

// Exécution séquentielle puis parallèle
async function demonstration() {
  console.time("séquentiel");
  await envoyerMessagesSequentiels(canaux, message);
  console.timeEnd("séquentiel"); // ~2.5 secondes (5 x 500ms)

  console.log("\n---\n");

  console.time("parallèle");
  await envoyerMessagesParalleles(canaux, message);
  console.timeEnd("parallèle"); // ~500ms (tous en même temps)
}

demonstration();
```

## Gestion d'erreurs en asynchrone

La gestion des erreurs est cruciale dans le développement de bots pour assurer leur robustesse.

### Avec les Promises

```javascript
function operationRisquee() {
  return new Promise((resolve, reject) => {
    const reussite = Math.random() > 0.5;

    setTimeout(() => {
      if (reussite) {
        resolve("Opération réussie");
      } else {
        reject(new Error("L'opération a échoué"));
      }
    }, 500);
  });
}

// Gestion d'erreur avec .catch()
operationRisquee()
  .then((resultat) => {
    console.log("Succès:", resultat);
  })
  .catch((erreur) => {
    console.error("Une erreur est survenue:", erreur.message);
    // Possibilité de récupération ou de fallback ici
    return "Valeur de secours";
  })
  .then((valeur) => {
    console.log("Traitement continué avec:", valeur);
  });
```

### Avec async/await

```javascript
async function executerOperationSecurisee() {
  try {
    const resultat = await operationRisquee();
    console.log("Opération réussie:", resultat);
    return resultat;
  } catch (erreur) {
    console.error("Erreur capturée:", erreur.message);
    // Récupération ou fallback
    return "Valeur alternative";
  } finally {
    console.log("Nettoyage - exécuté qu'il y ait erreur ou non");
  }
}

executerOperationSecurisee().then((valeurFinale) => {
  console.log("Traitement terminé avec:", valeurFinale);
});
```

## Application concrète : création d'un mini-bot Discord

Voici comment ces concepts asynchrones s'appliquent dans un mini-bot Discord (code conceptuel) :

```javascript
// Note: Ce code est conceptuel et nécessitera l'installation de discord.js
// npm install discord.js

const { Client, GatewayIntentBits } = require("discord.js");
const fs = require("fs").promises; // Version Promise de fs

// Création du client Discord
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ],
});

// Fonction asynchrone pour charger la configuration
async function chargerConfig() {
  try {
    // Lecture de fichier asynchrone
    const data = await fs.readFile("./config.json", "utf8");
    return JSON.parse(data);
  } catch (erreur) {
    console.error("Erreur lors du chargement de la configuration:", erreur);
    // Configuration par défaut en cas d'erreur
    return { prefix: "!", token: null };
  }
}

// Fonction asynchrone pour charger les commandes
async function chargerCommandes() {
  const commandes = new Map();

  try {
    // Lecture du dossier des commandes
    const fichiers = await fs.readdir("./commandes");

    // Filtrer uniquement les fichiers .js
    const fichierCommandes = fichiers.filter((fichier) =>
      fichier.endsWith(".js")
    );

    // Charger chaque commande
    for (const fichier of fichierCommandes) {
      const commande = require(`./commandes/${fichier}`);
      commandes.set(commande.nom, commande);
      console.log(`Commande chargée: ${commande.nom}`);
    }
  } catch (erreur) {
    console.error("Erreur lors du chargement des commandes:", erreur);
  }

  return commandes;
}

// Initialisation du bot
async function initialiserBot() {
  try {
    // Chargement parallèle de la config et des commandes
    const [config, commandes] = await Promise.all([
      chargerConfig(),
      chargerCommandes(),
    ]);

    if (!config.token) {
      throw new Error("Token Discord manquant dans la configuration");
    }

    // Stockage global
    client.config = config;
    client.commandes = commandes;

    // Gestionnaire d'événements de messages
    client.on("messageCreate", async (message) => {
      // Ignorer les messages des bots et ceux sans préfixe
      if (message.author.bot || !message.content.startsWith(config.prefix))
        return;

      // Extraire les arguments et le nom de la commande
      const args = message.content
        .slice(config.prefix.length)
        .trim()
        .split(/ +/);
      const commandName = args.shift().toLowerCase();

      // Vérifier si la commande existe
      if (!commandes.has(commandName)) return;

      // Exécuter la commande
      try {
        await commandes.get(commandName).execute(message, args);
      } catch (erreur) {
        console.error(`Erreur dans la commande ${commandName}:`, erreur);
        await message.reply(
          "Une erreur s'est produite lors de l'exécution de cette commande."
        );
      }
    });

    // Connexion au serveur Discord
    await client.login(config.token);
    console.log(`Bot connecté en tant que ${client.user.tag}`);
  } catch (erreur) {
    console.error("Erreur fatale lors de l'initialisation du bot:", erreur);
    process.exit(1);
  }
}

// Démarrer le bot
initialiserBot();

// Gestionnaire d'erreurs non gérées pour la stabilité
process.on("unhandledRejection", (erreur) => {
  console.error("Erreur non gérée:", erreur);
});
```

## Exercice pratique : Timer de quiz asynchrone

Créons un système de quiz asynchrone que vous pourriez intégrer dans un bot :

```javascript
// quiz-system.js
class QuizSystem {
  constructor() {
    this.questions = [
      {
        question: "Quel est le nom du moteur JavaScript utilisé par Node.js ?",
        reponse: "v8",
      },
      {
        question:
          "Quelle méthode permet d'attendre plusieurs promesses en parallèle ?",
        reponse: "promise.all",
      },
      {
        question: "Quel mot-clé permet de définir une fonction asynchrone ?",
        reponse: "async",
      },
    ];
    this.quizEnCours = false;
    this.questionActuelle = null;
    this.timerQuestion = null;
    this.scores = new Map();
  }

  // Démarrer un nouveau quiz
  async demarrerQuiz(canal, dureeSecondes = 30) {
    if (this.quizEnCours) {
      throw new Error("Un quiz est déjà en cours");
    }

    this.quizEnCours = true;
    this.scores.clear();

    console.log(`Démarrage d'un quiz de ${this.questions.length} questions...`);
    await this.simuleEnvoiMessage(canal, "📝 **NOUVEAU QUIZ** 📝");
    await this.simuleEnvoiMessage(
      canal,
      "Préparez-vous ! Le quiz commence dans 3 secondes..."
    );

    // Pause avant démarrage
    await this.pause(3000);

    // Poser chaque question séquentiellement
    for (let i = 0; i < this.questions.length; i++) {
      if (!this.quizEnCours) break; // Au cas où le quiz serait arrêté en cours

      const question = this.questions[i];
      const reponseObtenue = await this.poserQuestion(
        canal,
        question,
        dureeSecondes
      );

      // Pause entre les questions
      if (i < this.questions.length - 1) {
        await this.pause(2000);
      }
    }

    // Fin du quiz
    if (this.quizEnCours) {
      this.quizEnCours = false;
      await this.afficherResultats(canal);
    }
  }

  // Poser une question et attendre la réponse ou le timeout
  async poserQuestion(canal, question, dureeSecondes) {
    return new Promise(async (resolve) => {
      this.questionActuelle = question;

      await this.simuleEnvoiMessage(
        canal,
        `⏱️ **Question (${dureeSecondes}s) :** ${question.question}`
      );

      // Simuler le timer
      console.log(`[Timer] Démarrage timer de ${dureeSecondes} secondes`);
      let tempsRestant = dureeSecondes;

      // Afficher le temps restant tous les 5 secondes
      const intervalAffichage = setInterval(() => {
        if (tempsRestant % 5 === 0 && tempsRestant > 0) {
          this.simuleEnvoiMessage(
            canal,
            `⏱️ ${tempsRestant} secondes restantes...`
          );
        }
      }, 1000);

      // Définir le timeout principal
      this.timerQuestion = setTimeout(async () => {
        clearInterval(intervalAffichage);
        await this.simuleEnvoiMessage(
          canal,
          `⌛ **Temps écoulé !** La réponse était: ${question.reponse}`
        );
        this.questionActuelle = null;
        resolve(null);
      }, dureeSecondes * 1000);

      // Simuler des réponses aléatoires pour la démonstration
      this.simuleReponses(canal, question, dureeSecondes).then(
        (utilisateur) => {
          clearTimeout(this.timerQuestion);
          clearInterval(intervalAffichage);
          this.questionActuelle = null;
          resolve(utilisateur);
        }
      );
    });
  }

  // Simuler des réponses d'utilisateurs (pour la démonstration)
  async simuleReponses(canal, question, dureeSecondes) {
    return new Promise((resolve) => {
      const utilisateursTest = ["Alice", "Bob", "Charlie", "Dave", "Eve"];
      const reponsesPossibles = [
        question.reponse, // Réponse correcte
        question.reponse.toUpperCase(), // Variante correcte
        "je ne sais pas",
        "aucune idée",
        question.reponse + "s", // Presque correct
        "autre réponse",
      ];

      // Délai aléatoire pour chaque utilisateur
      utilisateursTest.forEach((utilisateur) => {
        // 70% de chance de répondre à cette question
        if (Math.random() < 0.7) {
          const delai = Math.floor(
            Math.random() * (dureeSecondes * 0.9) * 1000
          );
          const reponseIndex = Math.floor(
            Math.random() * reponsesPossibles.length
          );
          const reponse = reponsesPossibles[reponseIndex];

          setTimeout(() => {
            // Ne pas répondre si le quiz est terminé entre-temps
            if (!this.questionActuelle) return;

            this.verifierReponse(canal, utilisateur, reponse);

            // Si c'est la bonne réponse, résoudre la promesse
            if (this.estReponseCorrecte(reponse, question.reponse)) {
              console.log(`[Quiz] ${utilisateur} a trouvé la bonne réponse`);
              resolve(utilisateur);
            }
          }, delai);
        }
      });
    });
  }

  // Vérifier si une réponse est correcte
  estReponseCorrecte(reponseUtilisateur, reponseAttendue) {
    return reponseUtilisateur.toLowerCase() === reponseAttendue.toLowerCase();
  }

  // Traiter la réponse d'un utilisateur
  async verifierReponse(canal, utilisateur, reponse) {
    if (!this.questionActuelle) return false;

    console.log(`[Quiz] ${utilisateur} répond: "${reponse}"`);

    const estCorrect = this.estReponseCorrecte(
      reponse,
      this.questionActuelle.reponse
    );

    if (estCorrect) {
      // Ajouter au score
      const scoreActuel = this.scores.get(utilisateur) || 0;
      this.scores.set(utilisateur, scoreActuel + 1);

      // Envoyer un message de félicitations
      await this.simuleEnvoiMessage(
        canal,
        `✅ **${utilisateur}** a trouvé la bonne réponse : ${this.questionActuelle.reponse}`
      );
      clearTimeout(this.timerQuestion);
      return true;
    }

    return false;
  }

  // Afficher les résultats finaux
  async afficherResultats(canal) {
    await this.simuleEnvoiMessage(canal, "🏁 **Fin du quiz !** 🏁");

    if (this.scores.size === 0) {
      await this.simuleEnvoiMessage(
        canal,
        "Personne n'a trouvé de bonne réponse. 😭"
      );
      return;
    }

    // Trier par score décroissant
    const classement = Array.from(this.scores.entries())
      .sort((a, b) => b[1] - a[1])
      .map(
        ([nom, score], index) => `${index + 1}. **${nom}** : ${score} point(s)`
      );

    await this.simuleEnvoiMessage(
      canal,
      "📊 **Classement final** 📊\n" + classement.join("\n")
    );

    // Féliciter le gagnant
    const gagnant = classement[0].split("**")[1];
    await this.simuleEnvoiMessage(
      canal,
      `🎉 Félicitations à **${gagnant}** qui remporte ce quiz !`
    );
  }

  // Arrêter le quiz en cours
  async arreterQuiz(canal) {
    if (!this.quizEnCours) return;

    clearTimeout(this.timerQuestion);
    this.quizEnCours = false;
    this.questionActuelle = null;

    await this.simuleEnvoiMessage(
      canal,
      "⚠️ Le quiz a été arrêté manuellement."
    );
  }

  // Méthodes utilitaires pour simuler un environnement de bot
  async simuleEnvoiMessage(canal, message) {
    console.log(`[${canal}] ${message}`);
    // Simuler un léger délai d'envoi de message
    await this.pause(Math.random() * 200 + 100);
  }

  async pause(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}

// Démo du système de quiz
async function demoQuiz() {
  const quiz = new QuizSystem();

  console.log("=== Démonstration du système de quiz asynchrone ===");

  // Tester le système de quiz
  try {
    await quiz.demarrerQuiz("général", 10); // 10 secondes par question
  } catch (erreur) {
    console.error("Erreur lors du quiz:", erreur);
  }

  console.log("=== Fin de la démonstration ===");
}

// Lancer la démo
demoQuiz();
```

## Concepts avancés

### Limiter les opérations parallèles

Dans certains cas, vous voudrez limiter le nombre d'opérations asynchrones simultanées, par exemple pour respecter les limites d'une API :

```javascript
// Fonction qui limite le nombre d'opérations parallèles
async function paralleleLimit(taches, limite) {
  // Tableau pour stocker les promesses en cours
  const promessesEnCours = [];
  // Tableau pour stocker tous les résultats
  const resultats = [];

  // Compteur pour suivre l'index de la prochaine tâche
  let index = 0;

  // Fonction pour exécuter une tâche et la gérer
  async function executerTache() {
    // Obtenir l'index actuel puis incrémenter
    const indexCourant = index++;

    // Si toutes les tâches sont lancées, ne rien faire
    if (indexCourant >= taches.length) return;

    try {
      // Exécuter la tâche et stocker son résultat
      const resultat = await taches[indexCourant]();
      resultats[indexCourant] = resultat;
    } catch (erreur) {
      resultats[indexCourant] = { erreur };
    }

    // Exécuter la tâche suivante quand celle-ci est terminée
    return executerTache();
  }

  // Lancer les tâches jusqu'à la limite
  const promessesDemarrage = [];
  for (let i = 0; i < Math.min(limite, taches.length); i++) {
    promessesDemarrage.push(executerTache());
  }

  // Attendre que toutes les tâches soient terminées
  await Promise.all(promessesDemarrage);

  return resultats;
}

// Exemple d'utilisation
async function testParalleleLimit() {
  console.log("Test de limitation des opérations parallèles");

  // Créer des tâches qui simulent des appels API
  const nbTaches = 10;
  const taches = Array(nbTaches)
    .fill(null)
    .map((_, i) => {
      return async () => {
        console.log(`Démarrage de la tâche ${i + 1}`);
        // Simuler un temps de traitement variable
        const delai = Math.floor(Math.random() * 2000 + 500);
        await new Promise((resolve) => setTimeout(resolve, delai));
        console.log(`Fin de la tâche ${i + 1} (${delai}ms)`);
        return { index: i + 1, delai };
      };
    });

  console.time("execution");

  // Exécuter avec une limite de 3 tâches simultanées
  const resultats = await paralleleLimit(taches, 3);

  console.timeEnd("execution");
  console.log("Résultats:", resultats);
}

// Lancer le test
testParalleleLimit();
```

### Timeout sur les opérations asynchrones

Pour éviter qu'une opération ne bloque trop longtemps, vous pouvez ajouter un timeout :

```javascript
// Fonction pour ajouter un timeout à n'importe quelle promesse
function avecTimeout(promesse, ms, message = "L'opération a expiré") {
  // Créer une promesse qui rejette après le délai
  const timeout = new Promise((_, reject) => {
    setTimeout(() => {
      reject(new Error(message));
    }, ms);
  });

  // Utiliser Promise.race pour voir laquelle termine en premier
  return Promise.race([promesse, timeout]);
}

// Exemple d'utilisation
async function testTimeout() {
  console.log("Test des timeouts sur opérations asynchrones");

  // Fonction qui simule une opération lente
  async function operationLente() {
    console.log("Opération lente démarrée...");
    await new Promise((resolve) => setTimeout(resolve, 3000));
    console.log("Opération lente terminée");
    return "Résultat de l'opération lente";
  }

  // Fonction qui simule une opération rapide
  async function operationRapide() {
    console.log("Opération rapide démarrée...");
    await new Promise((resolve) => setTimeout(resolve, 500));
    console.log("Opération rapide terminée");
    return "Résultat de l'opération rapide";
  }

  try {
    // Tester avec l'opération rapide (devrait réussir)
    console.log("Test avec opération rapide (timeout 2s):");
    const resultat1 = await avecTimeout(
      operationRapide(),
      2000,
      "L'opération rapide a pris trop de temps"
    );
    console.log("Résultat:", resultat1);

    console.log("\n---\n");

    // Tester avec l'opération lente (devrait échouer avec timeout)
    console.log("Test avec opération lente (timeout 2s):");
    const resultat2 = await avecTimeout(
      operationLente(),
      2000,
      "L'opération lente a pris trop de temps"
    );
    console.log("Résultat:", resultat2);
  } catch (erreur) {
    console.error("Erreur capturée:", erreur.message);
  }
}

// Lancer le test
testTimeout();
```

## Conclusion

Dans ce chapitre, vous avez appris :

- La nature asynchrone de JavaScript et son importance pour les bots
- Comment utiliser les callbacks, les Promises et async/await
- Comment gérer plusieurs opérations asynchrones en parallèle ou en séquence
- Comment gérer les erreurs dans un contexte asynchrone
- Comment appliquer ces concepts à des cas concrets de développement de bots

Ces concepts sont fondamentaux pour développer des bots Discord et Twitch performants et réactifs. Dans le prochain chapitre, nous explorerons la structure des projets Node.js et l'organisation du code pour des bots plus complexes.

## Ressources supplémentaires

- [JavaScript Promises: an Introduction](https://web.dev/promises/)
- [Asynchronous Programming in JavaScript](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous)
- [JavaScript Async/Await Tutorial](https://javascript.info/async-await)
- [Node.js Event Loop Explained](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
