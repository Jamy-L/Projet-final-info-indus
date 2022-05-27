## Jeu sur deux cartes STM32 liées par SPI
Ce projet met en oeuvre un jeu se jouant entre deux cartes STM32 connectée par SPI. Il comprend deux programmes: le programme "Master" est celui de la carte du joueur 1, et le programme "Slave" celui du joueur 2. Le programme "Master" collecte les input du joueur 1, et récupère les inputs du joueur 2 via la liaison SPI. Il met aussi à jour les données du jeu (position de l'ennemi, score des joueurs), et affiche les données à l'écran. Il envoie aussi ces données au joueur 2 via la liaison SPI.
Le programme Slave récupère les inputs du joueur 2 et les envoie au joueur 1, tout en recevant les données de jeu. Il les affiche ensuite.

![Projet Final](https://user-images.githubusercontent.com/46826148/170218525-9678f982-c12a-495a-99b1-1b6d0b76a9b4.png)
### Les blackboards
Les programmes "Master" et "Slave" comportent chacun un blackboard soumis à un mutex. Ces derniers contiennent les données du jeu: position de l'ennemi, position du viseur du joueur 1 et du joueur 2, score des deux joueurs.
### Les interruptions
Une interruption matérielle est déclenchée par pression du bouton "BP1", dans le programme "Master" et dans le programme "Slave". Cette derniere envoie un message à la tâche "Tir" par une messagerie, afin de notifier de l'appui.

Une interruption est aussi déclenchée lorsqu'une transission SPI est terminée. Au niveau du Slave, cette interruption rappelle directement une transmissions. Au niveau du Master, elle notifie la tâche "Share" via une messagerie.
### Les tâches
Le programme "Master" comporte plusieurs tâches:
<ul>
  <li> "Viseur" collecte les données de l'ADC afin de convertir l'input du Joystick du joueur 1 en une position pour le viseur. Elle s'accapare le mutex pour écrire cette position dans le blackboard Master. Cette tâche est périodique</li>
  <li> "Affichage" se déclenche périodiquement, et affiche l'ennemi (un disque bleu), ainsi que les viseurs des joueurs 1 et 2. Elle d'accapre le mutex du blackboard Master</li>
  <li> "Tir" est executée périodiquement mais ne se lance réellement que lorsque l'appui sur le bouton BP1 a été notifié par interruption via une messagerie. Cette tâche vérifie alors si le joueur 1 a bien touché l'énnemi. Si c'est le cas, l'ennemi est notifié via une file d'attente.</li>
  <li> "Ennemi" est executée périodiquement et gère la position de l'ennemi. S'il n'est pas touché par les joueurs, il se déplace selon sa direction de mouvement. La direction est mise à jour lorsqu'il rebondit sur les bords de l'écran. La tâche "Ennemi" est notifiée via une file d'attente si l'ennemi est touché ou non, et par quel joueur. Si il est touché, le score du joueur est augmenté, et la tâche "Ennemi" est supprimée puis une nouvelle est crée. Lors de l'initialisation d'un ennemi, les coordonnées sont tirées aléatoirement à l'aide du RNG.</li>
  <li> "Share" est appelée périodiquement et gère la communication avec le Slave. La fonction <code>HAL_SPI_TransmitReceive_IT</code> est appelée, et la tâche sera notifiée par une messagerie lorsque la réception est terminée. Après réception, les donées reçues sont stockées sur le blackboard.</li>
</ul>
Le programme "Slave" fonctionne de presque de la même manière, néanmoins:
<ul>
  <li>La tâche "Ennemi" n'existe pas puisque l'ennemi est géré par le programme Master et est envoyé par SPI au programme Master.</li>
  <li>Lorsque le joueur 2 touche l'ennemie, la tâche "Tir" envoie ainsi directement l'information à la tâche "Share" via une file d'attente, afin d'envoyer l'information par SPI.  C'est la tâche "Ennemi" du programme Master qui gérera la suite.</li>
