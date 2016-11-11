# Les Zones

Maintenant qu'`Angular2` est officiellement disponible, il est temps pour nous tous de comprendre les méandres de cette
toute nouvelle version du framework. Si vous avez eu un jour la curiosité de faire un petit tutorial *Getting Started*
pour débuter un nouveau projet `Angular`, vous avez sûrement dû importer une librairie `zone.js`, afin de faire fonctionner
l'application correctement. Vous l'avez importée, mais vous ne savez peut-être pas à quoi elle sert. Nous allons essayer 
d'y répondre dans cet article. 

Avant de vous présenter la librairie en elle-même, nous souhaitons tout d'abord vous présenter deux problèmes qui 
pourront être résolus grâce aux zones. 

## Calcul du temps d'exécution d'un script JS

Le code source JavaScript ci-dessous a pour but de calculer le temps d'exécution d'un code applicatif. 
Le lancement du timer est réalisé par la méthode `startTimer` et est arrété dans la méthode `endTimer`. 
Nous pouvons imaginer également que nous affichons le résultat de ce timer via un `console.log` directement
dans la méthode `endTimer`. Mais les implémentations de ces deux méthodes ne sont en fait pas intéressantes pour 
le reste de cet article. 

```JavaScript
startTimer()
goToZenikaParis();
setTimeout( _ => {
	veryLongTask()
}, 0);
setTimeout( _ => {
	veryLongTask()  
}, 0);
goBackToZenikaLille();
endTimer()
```

Ce script est composé de traitements synchrones et de traitements asynchrones (les deux appels
à la méthode `setTimeout`). Pour les débutants en `JavaScript`, on pourrait imaginer que du fait du deuxième
paramètre de mes méthodes `setTimeout` qui est égal à 0, la méthode `veryLongTask` est appelée deux fois, 
l'une après l'autre, de manière synchrone. Ce serait une erreur...

Si nous nous mettons à la place d'une VM `JavaScript`, nous avons en fait, au minimum, deux stacks d'exécution. La principale correspond
à celle que nous avons l'habitude de manipuler, celle qui va exécuter nos traitements synchrones. Dès que la VM détecte
un appel à un traitement asynchrone (`setTimeout`, `setInterval`, `addEventListener`, ...), elle ne l'exécute pas de suite, mais place
l'appel à cette méthode dans la seconde stack. Et cette dernière ne sera dépilée que lorsque la stack principale
sera vide. 

Dans notre exemple, après avoir exécuté la méthode `endTimer`, la stack principale est vide, et la stack secondaire contenant
les deux appels à la méthode `setTimeout` vont enfin pouvoir être exécutés. Et vous comprenez maintenant le problème : nous affichons
le résultat de notre timer avant même la fin de l'ensemble de notre code applicatif. Le résultat sera alors erroné. 


## Gestion des stacktraces

Le deuxième exemple que nous souhaitons vous présenter concerne la gestion des stacktraces d'erreur, 
lors de l'exécution d'un ensemble de traitements asynchrones. 

Dans l'exemple ci-dessous, lorsque l'erreur est émise dans la méthode `throwError`, dans notre console, nous 
n'avons aucune information sur l'origine réelle de cette erreur. Nous ne savons qu'elle a été émise parce que 
nous avons cliqué sur le bouton *Cause Error*, et également parce que nous avons cliqué au préalable sur le 
bouton *Bind Error* 

```html
<html>
  <body>  
    <button id="b1">Bind Error</button>
    <button id="b2">Cause Error</button>

    <script>
      function main () {
        b1.addEventListener('click', bindSecondButton);
      }

      function bindSecondButton () {
        b2.addEventListener('click', throwError);
      }

      function throwError () {
        throw new Error('aw shucks');
      }

      main();
    </script>
  </body>
</html>
```

Ce code est tiré de la bibliothèque d'exemples fournie dans le repository Github de la librairie. 

## La solution: Zone.JS

Cette librairie va nous permettre de créer des contextes d'exécution, dans lesquels nous allons pouvoir 
faire exécuter des traitements synchrones et asynchrones. Ces contextes d'exécution vont nous permettre de
faire deux choses : 

* Partager des données à travers les différents traitements asynchrones exécutés dans le même contexte. 
* Interagir avec le cycle de vie des tâches asynchrones, à l'aide de *hooks* 

Comment est-il possible d'interagir avec le cycle de vie de ces tâches ? Tout simplement parce que la librairie
surcharge l'ensemble des traitements asynchrones du langage dans le but de pouvoir exécuter les *hooks* que 
nous aurions configurés. 

Afin de vous présenter la syntaxe pour manipuler cette librairie, nous allons reprendre le deuxième problème 
présenté précédemment. Une fois la librairie `zone.js` importée, 
vous allez avoir accès à un objet `Zone.current` qui est le contexte d'exécution par défaut. A partir de ce contexte, 
nous allons pouvoir en créer de nouveau grâce à la méthode `fork`. Cette dernière retourne le même type d'objet, 
une zone. Nous pouvons appeler à nouveau la même méthode `fork` sur cette zone, afin d'avoir une hiérarchie dans nos contextes
d'exécution, et de pouvoir partager du code commun. Sur l'objet retourné par la méthode `fork`, nous pouvons
à présent appeler notre code métier grâce à la méthode `run`.

```html
<html>
  <body>  
    <button id="b1">Bind Error</button>
    <button id="b2">Cause Error</button>
    
    <script src="zone.js"></script>
    <script>
      function main () {
        b1.addEventListener('click', bindSecondButton);
      }

      function bindSecondButton () {
        b2.addEventListener('click', throwError);
      }

      function throwError () {
        throw new Error('aw shucks');
      }

      Zone.current.fork({}).run(main);
    </script>
  </body>
</html>
```

La méthode `fork` prend un paramètre correspondant aux `hooks` que nous désirons exécuter aux différentes
étapes du cycle de vie de nos traitements asynchrones. Nous n'allons pas présenter tous les `hooks` possibles, 
dans notre cas d'utilisation, nous allons définir des implémentations pour les suivants : 

* `onScheduleTask` : exécuté lorsqu'un traitement est placé dans la stack secondaire
* `onHandleError` : lorsqu'une erreur est émise

```JavaScript
Zone.current.fork({
  onScheduleTask: function (parentZoneDelegate, currentZone, targetZone , task) {
    
  },
  onHandleError: function (parentZoneDelegate, currentZone, targetZone, error) {
    
  }
}).run(main);
```

Nous avons indiqué précédemment que nous pouvions avoir une hiérarchie de zones. Cela se fait via l'appel de l'implémentation de la zone parente. L'appel se fait sur chaque *hook* et permet de de bénéficier correctement de la chaîne. Sans cette action la chaîne serait tout simplement brisée.

```JavaScript
Zone.current.fork({
  onScheduleTask: function (parentZoneDelegate, currentZone, targetZone , task) {
    parentZoneDelegate.scheduleTask(targetZone, task);
  },
  onHandleError: function (parentZoneDelegate, currentZone, targetZone, error) {
    parentZoneDelegate.handleError(targetZone, error);
  }
}).run(main);
```

Pour chaque traitement asynchrone, nous allons sauvegarder dans une variable disponible depuis notre contexte
d'exécution (`targetZone`) une référence sous la forme d'une `Erreur`. Cette variable correspondra à un tableau, dans 
lequel le dernier traitement asynchrone enregistré sera en haut de la pile (fonctionnement normal d'une *stacktrace*). Dans la solution
ci-dessous, nous utilisons un identifiant généré par la librairie `task.source`, mais nous pourrions être plus spécifiques et récupérer 
le nom de la méthode, sa signature, ou pourquoi pas le numéro de ligne, etc.

```JavaScript
Zone.current.fork({
  onScheduleTask: function (parentZoneDelegate, currentZone, targetZone , task) {
    let traces = targetZone.traces || [];
    targetZone.traces = [new Error(task.source)].concat(traces);
    parentZoneDelegate.scheduleTask(targetZone, task);
  },
  onHandleError: function (parentZoneDelegate, currentZone, targetZone, error) {
    parentZoneDelegate.handleError(targetZone, error);
  }
}).run(main);
```

La dernière chose à implémenter correspond à l'affichage de cette variable `targetZone.traces`, lorsqu'une 
vraie erreur est émise. Cette fonctionnalité sera réalisée dans la méthode `onHandleError`.
Nous allons tout d'abord ajouter à notre variable `targetZone.traces` l'erreur qui vient d'être émise, et nous allons 
ensuite surcharger le *getter* de la propriété `stack` du paramètre `error` de notre méthode. Ce *getter* retournera
nos différentes erreurs, séparées par des sauts de lignes. 

```JavaScript
Zone.current.fork({
  onScheduleTask: function (parentZoneDelegate, currentZone, targetZone , task) {
    let traces = targetZone.traces || [];
    targetZone.traces = [new Error(task.source)].concat(traces);
    parentZoneDelegate.scheduleTask(targetZone, task);
  },
  onHandleError: function (parentZoneDelegate, currentZone, targetZone, error) {
    targetZone.traces = [new Error(error.stack)].concat(targetZone.traces);
    Object.defineProperty(error, 'stack', {
        get(){
            return targetZone.traces.map(trace => trace.stack).join('\n');
        }
    });
    console.error(error.stack)
    parentZoneDelegate.handleError(targetZone, error);
  }
}).run(main);
```

Si nous affichons la valeur de cette variable `error.stack`, nous avons à présent toutes les informations 
nécessaires pour corriger le problème efficacement. 

```
index.html:63 Error: Error: aw shucks
    at HTMLButtonElement.throwError (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:40:11)
    at e.invokeTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:15126)
    at n.runTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:12486)
    at HTMLButtonElement.invoke (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:16239)
    at Object.onHandleError (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:55:28)
    at e.handleError (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:14595)
    at n.runTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:12540)
    at HTMLButtonElement.invoke (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:16239)
Error: HTMLButtonElement.addEventListener:click
    at Object.onScheduleTask (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:51:28)
    at e.scheduleTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:14742)
    at n.scheduleEventTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:12924)
    at file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:2031
    at HTMLButtonElement.addEventListener (eval at l (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:3113), <anonymous>:3:43)
    at HTMLButtonElement.bindSecondButton (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:35:8)
    at e.invokeTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:15126)
    at n.runTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:12486)
    at HTMLButtonElement.invoke (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:16239)
Error: HTMLButtonElement.addEventListener:click
    at Object.onScheduleTask (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:51:28)
    at e.scheduleTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:14742)
    at n.scheduleEventTask (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:12924)
    at file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:2031
    at HTMLButtonElement.addEventListener (eval at l (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:3113), <anonymous>:3:43)
    at main (file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:22:8)
    at e.invoke (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:14497)
    at n.run (file:///home/manu/Documents/workspaces/oss/demo-zone/zone.js:1:11884)
    at file:///home/manu/Documents/workspaces/oss/demo-zone/index.html:66:6
```
## Utilisation dans Angular2

Maintenant que nous avons vu en détail le fonctionnement de la librairie, vous
vous posez peut-être la question de son utilité dans `Angular2`, et voulez savoir dans quels cas le
framework a besoin des zones pour fonctionner. Pour ne citer que deux usages : 

* la détection des fins de traitements aynchrones pour faire la mise à jour de nos vues (il n'est
plus nécessaire d'utiliser des `$scope.$apply` ou `$scope.$digest` comme dans `AngularJS`)

* Avoir des *stacktraces* complètes (similaires à celles que nous avons créées précédemment), lors d'une 
erreur dans votre application (erreur dans votre code `TypeScript` ou dans vos *templates*). 


Si `Angular2` s'arrêtait là, nous pourrions avoir un léger problème de performance. Si nous avions une
synchronisation de vos vues après chaque traitement asynchrone, la performance pourrait être dégradée
surtout si nous utilisons des animations, des évènements de la souris, ou encore l'envoi
de requêtes HTTP qui ne nécessitent pas de mise à jour (des requêtes vers Google Analytics par exemple). 
Afin de remédier à ce problème, `Angular2` met à disposition un service, `NgZone`, qui est juste un wrapper sur 
l'objet `Zone`, avec notamment une méthode `runOutsideAngular` permettant de désactiver le fonctionnement par 
défaut. 

L'exemple ci-dessous permet d'envoyer une requête HTTP sans avoir besoin de mettre à jour vos vues lorsque
le serveur aura envoyé la réponse. 

```TypeScript
export class AnalyticsService {
	constructor(private zone:NgZone, private http: Http){}
	sendToAnalitics(stats){
 		this.zone.runOutsideAngular(() => { 
          this.http.post(“/api/analytics”, stats);
      });
	}
}
```

Cette fonctionnalité des `Zones` a été proposée par l'équipe `Angular` au comité en charge de la spécification
du langage `JavaScript`. C'est encore le tout début (la proposition est à l'état *stage 0*), mais nous 
pourrions peut-être avoir un jour cette fonctionnalité nativement dans le langage. Ce qui indique bien 
que les standards évoluent, notamment grâce aux multiples projets/librairies/frameworks que nous aimons utiliser. 
