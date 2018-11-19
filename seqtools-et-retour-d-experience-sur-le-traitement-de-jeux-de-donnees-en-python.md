# SeqTools

SeqTools est une librairie que j'ai créée pour transformer des jeux de données trop gros pour tenir en mémoire. Elle remplit un rôle comparable à [itertools](https://docs.python.org/3/library/itertools.html) de la librairie standard, mais fait aussi en sorte de donner accès aux éléments par indexation, ce qui est plus pratique.

[Dépôt du code](https://github.com/nlgranger/SeqTools)
[Documentation](https://seqtools-doc.readthedocs.io)

L'objectif principal consiste à prendre une ou plusieurs sources de données et de les combiner ou de modifier leurs éléments pour obtenir une version transformée.
Par exemple, partant d'une liste de noms de fichiers d'images, on souhaite les charger, les redimensionner puis les passer en noir et blanc.

Pour essayer de rendre la présentation plus intéressante, je vais essayer de présenter deux aspects de la librairie qui peuvent intéresser un plus large public.

## Évaluation à la demande

Mon travail, c'est l'apprentissage statistique sur des vidéos, donc il me faut un moyen simple et rapide pour définir et de tester des transformations à appliquer sur ces vidéos pour extraire des données utiles.
Pour faciliter la manipulation de gros jeux de données et pour accéder rapidement à quelques valeurs, SeqTools utilise principalement l’exécution _à la demande_ (=[évaluation paresseuse](https://fr.wikipedia.org/wiki/%C3%89valuation_paresseuse) ou [lazy evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)).
Cela signifie simplement que les opérations que l'on applique sur le jeu de données ne sont appliquées que quand on réclame l'accès à un élément, et que les calculs sont fait uniquement pour l'élément souhaité en ignorant les autres.
Cette approche n'est pas nouvelle, je rappelle juste ses avantages :

- Utilisation mémoire minimale, on ne stocke aucun résultat intermédiaire.
- Possibilité de définir toute la chaîne de transformation rapidement...
- ... et d'accéder à n'importe quel élément (même un résultat intermédiaire) sans attendre que les calculs soit appliqués à l'ensemble des données.

En pratique, ça ressemble à ça :

```python
>>> def f(x):
...     print("-> calcul")
...     return x + 2
...
>>> a = [1, 2, 3, 4]
>>> m = seqtools.smap(f, a)
>>> # jusque là, f n'a pas servi, mais si on réclame un élément :
>>> m[0]
-> calcul
3
```

Au passage, l'indexation par tranches et l'itération sont prises en charge, donc c'est assez transparent pour l'utilisateur:

```python
>>> for v in m[:-2]:
...     print(v)
-> calcul
3
-> calcul
4
```

Bien sûr il y a un inconvénient majeur : les erreurs dans les transformations ne surgissent que quand un élément est appelé, ce qui rend le débogage difficile (ex : mais où ai-je demandé cette transformation ?).
Si vous vous heurtez au même problème un jour, je suggère d'utiliser la fonction [inspect.stack()](https://docs.python.org/3/library/traceback.html#traceback.extract_stack), qui retourne le fichier, le numéro de ligne et un extrait du code à cet emplacement pour toute la pile d'appel jusqu'à la fonction.
Je m'en sers pour enregistrer l'emplacement où est créée une transformation susceptible d'échouer.
En cas d'erreur plus tard, l'utilisateur reçoit un message pour l'aider à retracer l'origine du problème.

Plus précisément, j'utilise le mécanisme d'enchaînement d'erreurs (`raise ... from ...`) : toute erreur en provenance du code utilisateur (la fonction `f` dans l'exemple ci-dessus) est interceptée et renvoyée comme cause d'une exception générique qui contient les informations de débogage.

Le code simplifié ressemble à ça :

```python
try:
    return f(donnee[i])
except Exception as cause:  # interception de l'erreur
    msg = "erreur dans l'opération définie à {}".format(code_qui_a_créé_cet_object)
    raise EvaluationError(msg) from cause
```

## Évaluation asynchrone en arrière-plan

À un moment ou à un autre, il faut bien souvent appliquer les transformations à l'ensemble du jeu de donné, et si possible rapidement! Hélas, pour ceux qui ne sont pas familier à cet aspect de python, sachez que l'exécution concurrente et/ou sur plusieurs cœurs n'est pas son point fort. Voici donc un petit retour d'expérience.

Python propose deux stratégies : les threads et les processus [^1]. Dans les deux cas, ils nous servent à démarrer des fils d'exécution évaluent en arrière-plan les valeurs dont nous avons besoin. Les processus/threads communiquent leurs résultats de manière _asynchrone_ avec le script principal qui les a démarré. Je vous passe les joyeux détails d'ordonnancement des tâches (ex : si l'élément n°5 arrive avant l'élément n°4) car cette partie ne pose pas de problème autre que la logique, ce qui nous laisse les soucis techniques :

_Comment transférer les résultats de l'arrière plan vers l'utilisateur?_

Pour les threads, c'est assez facile puisque le thread python a accès à l'environnement du parent, il suffit donc d'assigner le résultat dans une liste par exemple.
Pour les processus, j'ai trouvé deux approches praticables :

- Utiliser l'objet [multiprocessing.SyncManager](https://docs.python.org/3/library/multiprocessing.html#multiprocessing.managers.SyncManager) qui ajoute une couche d'abstraction sur des queues, des verrous, etc. pour proposer un objet qui se manipule comme une liste depuis n'importe quel processus sans se poser de questions. C'est pratique et sûr vis-à-vis de la synchronisation et des accès concurrents, mais la synchronisation des données entre les processus repose sur de la sérialisation/dé-sérialisation ce qui induit un surcoût.
- Utiliser de la [mémoire partagée](https://docs.python.org/3/library/multiprocessing.html#sharing-state-between-processes) si les objets sont des tableaux de valeurs d'un type et d'un taille donnée.

J'ai utilisé la première pour ma fonction d'[évaluation anticipée](https://seqtools-doc.readthedocs.io/en/stable/reference.html#seqtools.prefetch) et la seconde pour [charger des buffers](https://seqtools-doc.readthedocs.io/en/stable/reference.html#seqtools.load_buffers) en mémoire.

_Comment arrêter le thread/processus ?_

C'est plus difficile qu'il n'y paraît! Au début, je pensais rester laxiste et laisser le collecteur de mémoire faire le ménage, ou pire laisser traîner jusqu'à la fin du script. Manque de bol, python garde une référence des threads/processus actifs [dans une liste](https://docs.python.org/3.6/library/threading.html#threading.enumerate), si bien que le [collecteur de mémoire bloque](https://stackoverflow.com/questions/49082914/python-weakref-finalize-not-run-if-background-threads-are-alive/49095183) lorsqu'il fait le ménage à la fin.

On peut utiliser le mode [démon](https://docs.python.org/3.6/library/threading.html#threading.Thread.daemon) pour détacher les workers de leur parent, mais j'ai eu peur de laisser des tâches zombies si le script principal plante.

Finalement, j'ai opté pour un système de signaux et de délais. Si un worker reste inoccupé trop longtemps, il notifie le script principal et s'arrête. Inversement, si le script parent se termine ou que l'objet qui contient les donnée est supprimé, on notifie l'arrêt. Le second cas est joliment traité par [weakref.finalize](https://docs.python.org/3.6/library/weakref.html#weakref.finalize) qui offre un vrai mécanisme de destructeur systématiquement évalué par le ramasse-miette, contrairement à la méthode [`__del__`](https://docs.python.org/3.6/reference/datamodel.html#object.__del__) sur les objets. Si le script parent plante ou que l'utilisateur l'interrompt brutalement, c'est moins drôle : on se heurte à des erreurs non documentées sur les queues (et par conséquent avec SyncManager). D'après mon expérience, ces erreurs dérivent toutes de `IOError`.

Par ailleurs, il semble que les processus enfants interceptent parfois le "CTRL-C" dans le terminal. Pour diriger correctement le signal vers le parent, la solution recommandée est de lancer le sous-processus ainsi :

```python
# on désactive le support pour SIGINT (CTRL-C/KeyboardInterrupt)
old_sig_hdl = signal.signal(signal.SIGINT, signal.SIG_IGN)
# on lance ensuite le sous-processus qui hérite du paramètre ci-dessus
workers.start()
# finalement, on restaure le support de SIGINT dans le script principal
signal.signal(signal.SIGINT, old_sig_hdl)
```

_Comment éviter que le thread/processus plante ?_
_Comment notifier l'utilisateur en cas d'erreur dans un thread ou un processus ?_

Même en supposant que SeqTools n'a aucune erreur (ce qui est, de toute évidence, vrai ;-)), ça n'empêche pas l'utilisateur d'appliquer une opération qui va planter. L'idée, c'est d'éviter de le punir pour ces erreurs et de faciliter le débogage.

Pour commencer, il faut barder de `try ... except` le code qui encapsule l'exécution du code utilisateur en arrière-plan... le flot d'exécution est assez complexe à mettre eu point pour un néophyte, même si le [résultat final](https://github.com/nlgranger/SeqTools/blob/master/seqtools/evaluation.py#L241-L287) peut sembler logique.

Du fait de l'exécution asynchrone, une erreur générée en appliquant une opération en arrière-plan peut survenir à n'importe quel moment. Pour faciliter la vie des utilisateurs, j'ai essayé de retarder la notification de cette erreur et de la renvoyer comme si elle venait d'arriver lorsque l'utilisateur réclame l'élément en question. Pour ce faire, mon code essaie de sérialiser l'exception soulevée et de la renvoyer au moment opportun. Il reste un petit écueil à passer : l'exception comporte la trace d'exécution qui n'est pas sérialisable. Heureusement, l'excellente bibliothèque tierce [tblib](https://pypi.org/project/tblib/) permet de nettoyer la trace des éléments problématiques tout en gardant un maximum d'informations pour le débogage.

Au final, l'interface est très simplifiée pour l'utilisateur, ça plante quand il faut!

```python
def f(x):
    time.sleep(1)
    return x + 2

donnees = [1, 2, 3, 4, None]
resultat = seqtools.smap(f, donnees)
resultat_rapide = seqtools.prefetch(resultat, nworkers=2, max_buffered=2)

for i in range(4):
    print(resultat_rapide[i])

# jusque là tout va bien, on obtient les résultats deux fois plus rapidement
# du moment qu'on les lit dans l'ordre.

resultat_rapide[-1]  # là ça renvoie une erreur qui dérive du problème de typage
```

Voilà, ce sera tout pour ce petit retour d'expérience avec python. Si ma librairie vous intéresse, je vous suggère de regarder le [tutoriel](https://seqtools-doc.readthedocs.io/en/stable/tutorial.html) et les [exemples](https://seqtools-doc.readthedocs.io/en/stable/examples.html) qui donnent une idée de ses possibilités.

[^1]: Je laisse de côté le mécanisme futures/async/await qui vient d'arriver avec python 3.7 car il n'est pas rétro-compatible, et incompatible avec les outils dont j'ai besoin.
