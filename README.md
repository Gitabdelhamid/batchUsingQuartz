# batchUsingQuartz
Exemple Quartz et Spring Scheduling

Quartz et Spring Scheduling
Quartz pour ceux qui ne le connaissent pas encore, est un ordonnanceur. Il permet de planifier des tâches pour des exécutions ponctuelles ou répétées. Les planifications possibles vont de la simple répétition infinie, à la répétition calendaire utilisant la syntaxe de cron (tous les jours à minuit, le 31 janvier 2009 à 12h00, …). Quartz est prévu pour toutes sortes d’applications allant des programmes standalone basiques aux gros systèmes JEE distribués.

De son côté, Spring scheduling nous donne le choix entre les Timers Java et Quartz. Avec ses interfaces, Spring nous permet de planifier, très facilement, des Jobs directement dans l’ApplicationContext. En revanche, la gestion des planifications dynamiques est moins évidente. Dans cet article, je vous présenterai l’API Quartz dans les grandes lignes, puis on mettra en place la planification Quartz avec Spring Scheduling.


Présentation de Quartz
Quartz est sorti de l’ombre en 2005 au moment de son intégration dans le projet OpenSymphony. C’est depuis devenu l’API de référence pour répondre à des besoins de planification et d’exécution de tâches répétitives. Fort de ses années d’existence et des nombreux retours utilisateurs, Quartz s’adresse à tout type d’application : aux petites applications dans une configuration minimale avec quelques Threads, comme aux gros systèmes avec des centaines de Threads répartis sur plusieurs machines.

Fonctionnalités
Planification fournissant un paramétrage très fin. On peut faire des tâches ponctuelles à une date précise, des tâches répétitives avec un nombre de répétitions paramétré et un délai entre chaque appel, ou bien utiliser une expression cron pour des planifications plus précises.
Séparation entre les tâches exécutables (« Jobs ») et les planifications (« Triggers »). On en tire une grande souplesse dans la gestion de l’ordonnancement.
Gestion d’erreur. Les Jobs peuvent remonter des JobExecutionException qui indiquent le comportement à adopter par le Scheduler. On peut relancer le Job immédiatement, supprimer le trigger qui l’a invoqué ou supprimer tous les Triggers. On peut aussi définir un comportement pour une planification ratée (ie. pas exécutée à la bonne date).
Jobs avec ou sans états. Quartz gère les données afférentes aux tâches dans un JobDataStore qui permet à chacune des tâches de récupérer des paramètres et/ou de modifier des données de son contexte. Pour chaque Job, on peut spécifier si les données doivent être maintenues entre chaque appel ou non. Notez bien qu’un Job StateFull ne peut s’exécuter en mode parallèle.
Persistance des Jobs. On peut configurer Quartz pour sauvegarder en base de données toutes les planifications, afin de pouvoir redémarrer l’application sans perte.
Mode client / serveur. Le Scheduler peut exposer ses services en RMI à destination d’autres Schedulers qui fonctionneront en proxy et reporteront leurs appels au serveur.
Support des transactions JTA. Quand la persistance est activée, on peut configurer l’ordonnanceur pour participer aux transactions.
Mise en cluster. En mode persistant, on peut configurer plusieurs Schedulers en cluster. Au moment où un Trigger doit se déclencher, un Scheduler pose un verrou dessus et le traite, ce qui répartit naturellement la charge. Pour la reprise sur erreur, quand une instance tombe, les autres vont récupérer les Jobs qui lui étaient affectés et feront tourner ceux marqués pour recouvrement (« request recovery »).
