# JRM's Syntax-rules Primer for the Merely Eccentric

En apprenant à écrire des macros Scheme, j'ai remarqué qu'il est
facile de trouver à la fois des exemples triviaux ou
extraordinairement complexes, mais il semble ne pas y en avoir de
difficulté intermédiaire. J'ai découvert quelques trucs en écrivant
des macros et quelques uns pourront les trouver utiles.

Le but de base d'une macro est l'abstraction *syntaxique*. Si les
fonctions vous permettent d'étendre la fonctionnalité du langage
Scheme sous-jacent, les macros vous permettent d'en étendre la
syntaxe. Une macro bien conçue peut grandement augmenter la
lisibilité d'un programme, mais une macro mal conçue peut rendre un
programme complètement illisible.

Les macros sont aussi souvent utilisées comme substituts de
fonction pour améliorer la performance. Dans un monde idéal ce
serait inutile, mais les compilateurs ont des limitations et une
macro peut parfois fournir un contournement. Dans ces cas, la macro
devrait être un remplacement 'drop-in' pour la fonction
équivalente, et le but en terme de conception ne serait alors pas
d'étendre la syntaxe du langage, mais de copier la syntaxe
existante autant que possible.

# Macros très simples.

SYNTAX-RULES donne accès à de très puissantes possibilités de
correspondance demotifs et de déstructuration. Avec des macros très
simples, cependant, la plus grande part de cette puissance est
inutilisée. Voici un exemple :

    (define-syntax nth-value
      (syntax-rules ()
        ((nth-value n values-producing-form)
         (call-with-values
           (lambda () values-producing-form)
           (lambda all-values
             (list-ref all-values n))))))
    
Lors de l'utilisation de fonctions qui retournent des valeurs
multiples, il arrive occasionnellement que vous ne vous intéressiez
qu'a une seule des valeurs retournées. La macro NTH-VALUE évalue un
VALUES-PRODUCING-FORM et en extrait la énième valeur de retour.

Avant que la macro ne soit évaluée, Scheme aurait traité une forme
commençant par NTH-VALUE comme il aurait traité tout autre forme :
il aurait recherché la valeur de la variable NTH-VALUE dans
l'environnement courant et l'aurait appliquée aux valeurs produites
par l'évaluation des arguments.

DEFINE-SYNTAX introduit une nouvelle forme spéciale pour Scheme.
Les formes qui commencent par NTH-VALUE ne sont plus de simples
applications de procédures. Quand Scheme traite une telle forme, il
utilise le SYNTAX-RULES que nous fournissons pour réécrire la
forme. La réécriture en résultant est alors traitée en lieu et
place de la forme originale.

*Les formes ne sont réécrites que si l'opérateur de position est un identifiant qui a été DEFINE-SYNTAXé. Les autres usages de cet identifiant ne sont pas réécrites.*

Vous ne pouvez pas écrire une macro 'infixe', pas plus que vous ne
pouvez écrire une macro dans une 'position emboitée', c'est à dire
qu'une expression comme ((foo x) y z) est toujours considérée comme
une application de procédure. (La sous-expression (foo x) sera
bien-sur réécrite mais elle ne sera pas en mesure d'affecter le
traitement des sousexpressions x et y). Ceci sera important par la
suite.

SYNTAX-RULES est basé sur le remplacement d'éléments. SYNTAX-RULES
définit une série de motifs et de templates. La forme est comparée
au motif et les différents éléments sont transcrites dans le
template. Ca semble assez simple, mais il y a une chose importante
à toujours avoir à l'esprit.

*LE SOUS-LANGAGE SYNTAX-RULES N'EST PAS DU SCHEME*

C'est un point crucial qu'il est facile d'oublier. Dans l'exemple
suivant :
    (define-syntax nth-value
      (syntax-rules ()
        ((nth-value n values-producing-form)
         (call-with-values
           (lambda () values-producing-form)
           (lambda all-values
             (list-ref all-values n))))))
    
le motif (nth-value n values-producing-form) ressemble à du code
Scheme et le template 
    (call-with-values
           (lambda () values-producing-form)
           (lambda all-values
             (list-ref all-values n)))
ressemble *réellement* à du code Scheme, mais quand le code Scheme
applique la réécriture syntax-rules il n'y a pas de SIGNIFICATION
SEMANTIQUE attachée aux éléments. La signification sera attachée
dans la suite du traitement, mais pas ici.

Une des raisons pour lesquelles il est facile de l'oublier est que
dans un grand nombre de cas, ça ne fait aucune différence, mais
quand vous écrivez des règles plus compliquées vous pouvez trouver
des expansions inattendues. Garder à l'esprit que syntax-rules ne
manipule que des motifs et des templates vous aidera à éviter les
confusions.

Cet exemple fait un bon usage des motifs et templates. Considérons
la forme : 
    (nth-value 1 (let ((q (get-number)) (quotient/remainder q d))

Durant l'expansion de nth-value, la sousfome
    (let ((q (get-number)) (quotient/remainder q d) 
*entière* est liée
à VALUES-PRODUCING-FORMS et est transcrite dans le template en

    (lambda () values-producing-form)

. 
L'exemple n'aurait pas marché si
VALUES-PRODUCING-FORM avait pu être lié qu'a un symbole ou un
nombre.

Un motif est constitué d'un symbole, d'une constante, d'une liste
(propre ou impropre), d'un vecteur de plusieurs patterns ou de
l'élément spécial "..." (Une série de points consécutifs dans ce
document signifiera *toujours* l'élément littéral "..." et ne sera
*jamais* utilisé pour une autre raison.) Il n'est pas permis
d'utiliser le même symbole deux fois dans un motif. Puisque le
motif est comparé à la forme, et puisque la forme débute *toujours*
par le mot-clé défini, elle ne participe pas à la comparaison.

Vous pouvez réserver certains symboles en tant que 'littéraux' en
les plaçant dans la liste qui est le premier argument de
syntax-rules. Fondamentalement, ils seront traités comme s'ils
étaient des constantes, mais il existe une subtilité qui permet de
les utiliser quand même comme variables. La subtilité 'fait ce
qu'il faut' alors je ne rentrerai pas dans les détails.

Vous pouvez trouver des macros écrites avec l'élément
"_" plutôt qu'en répétant le nom de la macro : 

    (define-syntax nth-value
        (syntax-rules ()
            ((_ n values-producing-form)
             (call-with-values
               (lambda () values-producing-form)
               (lambda all-values
                 (list-ref all-values n))))))
    

Personnellement, je trouve que ça peut être perturbant et je
préfère dupliquer le nom de la macro.

Voici les règles de la correspondance de motifs :

-   Un motif constant ne correspondra qu'a une constante qui lui
    est EQUAL?. Nous exploiterons ceci par la suite
-   Un symbole qui fait partie des 'littéraux' ne peut être comparé
    que par rapport à l'exact même symbole de la forme, et seulement si
    l'utilisateur de la macro ne l'a pas caché.
-   Un symbole qui n'est *pas* l'un des littéraux peut correspondre
    à *n’importe laquelle* des formes complètes. (Oublier ceci peut
    mener à des bugs surprenants).
-   Une liste propre de motifs ne peut correspondre qu'a une liste
    de formes de la même longueur et seulement si les sous motifs
    correspondent.
-   Une liste impropre de motifs ne peut correspondre à une forme
    liste d'une longueur identique ou plus grande seulement si les
    sous-motifs correspondent. La 'suite pointée' du motif sera mis en
    correspondance avec les éléments restants de la forme. Il y a
    rarement de sens à utiliser autre chose qu'un symbole dans la suite
    pointée du motif.
-   L'élément ... est spécial et sera discuté ultérieurement.

# Débugger les macros

Plus les macros se compliquées, plus elles deviennent complexes à
débugguer. La plupart des systèmes Scheme possèdent un mécanisme
par lequel vous pouvez invoquer l'expansion de macro sur un morceau
de structure de liste et revenir à la forme expansée. En MzScheme
vous pourriez faire ceci :
    (syntax-object->datum
           (expand '(nth-value 1 (let ((q (get-number)))
                                   (quotient/remainder q d)))))
    
En MIT Scheme :
    (unsyntax (syntax '(nth-value 1 (let ((q (get-number)))
                                          (quotient/remainder q d)))
                           (nearest-repl/environment)))
    
Préparez vous pour un résultat intéressant --- vous pouvez ne pas
réaliser combien de formes sont en fait des macros et combien de
code est produit. Le système de macros peut également reconnaître
et optimiser certains motifs d'utilisation de fonction. Il ne
serait pas inhabituel de voir que (caddr x) est expansé en (car (cdr (cdr x))) ou en (%general-car-cdr 6 x).

Préparez vous aussi à des constructions inexplicables. Des objets
syntaxiques peuvent faire référence à des liaisons qui ne sont
lexicalement visibles que depuis l'intérieur de l'expanseur. Des
objets syntaxiques peuvent contenir de l'information perdue quand
elles sont reconverties en structure de liste. Vous pouvez tomber
sur des expansions apparament illégales comme la suivante : (lambda
(temp temp temp) (set! a temp) (set! b temp) (set! c temp))

Il y a trois différents objets syntaxiques qui représentent les
trois paramètres différents du lambda terme, et chaque affectation
ne fait référence qu'a un seul, mais chaque objet syntaxique
particulier a le même nom symbolique, de sorte que leur identité
particulière est perdue quand ils sont transformés en structure
liste.

## truc de débugging

Un truc de débuggage très simple est d'enrober le template avec une
quote : 

    (define-syntax nth-value
          (syntax-rules ()
            ((_ n values-producing-form)
             '(call-with-values                    ;; Notez la quote!
                (lambda () values-producing-form)
                (lambda all-values
                  (list-ref all-values n))))))
    
Maintenant, la macro renvoie le template rempli comme une liste
quotée:

    (nth-value (compute-n) (compute-values))
    
    => (call-with-values (lambda () (compute-values))
         (lambda all-values (list-ref all-values (compute-n))))
    
Parfois, il est difficile de comprendre pourquoi un motif n'a pu
correspondre à quelque chose que vous pensez qu'il aurait du
correspondre, ou pourquoi il a correspondu à quelque chose auquel
il n'aurait pas du. Il est facile d'écrire une macro de test de
motif :
    (define-syntax test-pattern
      (syntax-rules ()
        ((test-pattern one two) "match 1")
        ((test-pattern one two three) "match 2")
        ((test-pattern . default) "fail")))
    
## truc de débugging

Un second truc est d'écrire un template de débuggage :

    (define-syntax nth-value
       (syntax-rules ()
         ((_ n values-producing-form)
          '("Debugging template for nth-value"
            "n is" n
            "values-producing-form is" values-producing-form))))
    
# Macros N-aires

Par l'utilisation d'une suite pointée dans le motif nous pouvons
écrire des macros qui prennent un nombre arbitraire d'arguments.

    (define-syntax when
      (syntax-rules ()
        ((when condition . body) (if condition (begin . body) #f))))

Un exemple de son utilisation est :

    (when (negative? x) (newline) (display "Bad number: negative."))

Le motif correspond comme suit :

condition = (negative? x)

body = ((newline) (display "Bad number: negative."))

Dans la mesure ou la variable motif ‘body’ est en position de suite
pointée, elle est mise en correspondance avec la liste des éléments
restants de la forme. Ceci peut mener à des erreurs inhabituelles.
Supposons que j’ai écrit la macro de cette façon :

    (define-syntax when
      (syntax-rules ()
        ((when condition . body) (if condition (begin body) #f))))
Le motif est mis en correspondance avec la liste des arguments
restants, de sorte qu’il s’expansera en une liste dans le template
:

    (when (negative? x) (newline) (display "Bad number: negative."))

s’expansera en : 

    (if (negative? x) (begin ((newline) (display "Bad number: negative."))) \#f)

Mais ça marche *presque*. La conséquence de la condition est
évaluée comme s’il s’agissait d’un appel de fonction. La “fonction”
est la valeur de retour de l’appel à newline et l’”argument” est la
valeur de retour de display. Les règles d’évaluation étant
d’évaluer les sous formes et d’après d’appliquer les résultats la
procédure résultante aux arguments résultants, ceci peut imprimer
en réalité une nouvelle ligne et afficher la chaine “Bad number:
negative.” avant de lever une erreur. On pourrait aisément se
tromper en pensant que la forme WHEN a abouti et que c’est le code
*découlant* au WHEN qui a levé l’erreur.

Le code originel avait ceci dans le template (begin . body).

Quand le template est rempli, le corps est placé dans la ‘suite
pointée’. Le corps étant une suite de formes, l’effet est similaire
à avoir utilisé cons plutôt que list pour créer la forme
résultante. Malheureusement, ce truc n’est pas généralisable; vous
ne pouvez que préfixer des choses de cette façon.

*Les ‘arguments restes’ de la macro sont liés à une liste de formes, alors souvenez vous de les ‘délister’ quelque part.*

Il y a un autre bug dans la forme de départ :

    (define-syntax when
        (syntax-rules ()
          ((when condition . body) (if condition (begin . body) #f))))
    
      (when (< x 5))

s'expand en

         (if (< x 5) (begin) \#f)

Souvenez-vous que cette variable de motif de correspond à tout, y
compris la liste vide. La variable de motif BODY est lié à la liste
vide. Quand la forme template (begin . body) est rempli, l'élément
BEGIN est consé à la liste vide résultant de la forme illégale
(BEGIN).

Puisque (when (< x 5)) est par essence inutilisable, une solution
est de modifier la macro comme ceci :

(define-syntax when (syntax-rules () ((when condition form . forms)
(if condition (begin form . forms) \#f))))

A présent le motif ne correspondra que si l'expression WHEN a au
moins une forme et il est garanti que la sous forme résultante
BEGIN contiendra au moins une forme.

Cette sorte de macro --- une macro qui prend un nombre arbitraire
de sous formes et les évalue au sein d'un 'begin implicite' --- est
extrêmement commune. Il est important de connaître cet idiome de
macro :

# Macro de Begin Implicite

Utilisez cet idiome pour une macro qui autorise un nombre
arbitrairement long de sous formes et les traite séquentiellement
(éventuellement en retournant la valeur de la dernière sous forme).
Le motif devrait se terminer en "FORM . FORMS)" pour assurer une
sous forme au minimum. Le template a soit (begin form . forms) or
utilise l'implicite begin d'une autre forme spéciale (lambda ()
form . forms)

## Un piège étrange et subtil

Supposons que vous êtes un ancien hacker en INTERCAL et que vous
regrettez profondément ce langage. Vous souhaitez écrire une macro
PLEASE qui se retire simplement de l'expansion :

    (please display "foo") =\> (display "foo")

Voici une tentative :

    (define-syntax please (syntax-rules () ((please . forms) forms)))

Ceci fonctionne sur quelques systèmes Scheme, mais pas sur d'autres
et la raison en est assez subtile. En Scheme, l'application de
fonction est indiquée syntaxiquement par une liste consistant d'une
fonction et de zero ou plusieurs arguments. La macro, bien qu'elle
retourne une telle liste, crée cette liste en tant que partie du
traitement de correspondance de motif. Quand une macro est
expansée, une attention particulière est apportée pour assurer que
les sous formes du point d'usage continuent à faire référence à
l'environnement lexical de ce point, tandis que les sous formes
sont introduites par le template continuent à faire référence à
l'environnement lexical de la définition de la macro. Cependant, la
liste qui est retournée par la macro PLEASE est une sous forme qui
n'a été créée *ni* au point d'utilisation de la macro, *ni* au
point de définition de la macro, mais en fait dans l'environnement
de la correspondance de motif. En MzScheme, cet environnement n'a
pas la correspondance syntaxique permettant d'interpréter une liste
comme appel de fonction, de sorte que l'erreur suivante se produit
: "compile: bad syntax; function application is not allowed,
because no \#%app syntax transformer is bound in: (display 33)"

La correction est triviale :

    (define-syntax please (syntax-rules () ((please function . arguments) (function . arguments))))

L'expansion résultante est a présent une liste construite à
l'intérieur du template de la macro, plutôt qu'une liste construite
par la correspondance de forme. L'environnement de template est
utilisé est la liste résultante est par conséquent interprétée
comme un appel de fonction.

Encore une fois, ceci est un point extrèmement subtil, mais il est
facile de se rappeler cette règle d'or :

*N'utilisez pas les arguments 'reste' de la macro comme un appel de fonction implicite. Utilisez un template avec un élément explicite (fonction . arguments).*

# Motifs multiples

Les règles de syntaxes permettent un nombre arbitraire d'arguments
de paires motif/template. Quand une forme doit être réécrite, une
correspondance au premier motif est tentée. Si le motif ne
correspond pas, le motif suivant est examiné. Le template associé
au premier motif correspondant est utilisé pour la réécriture. Si
aucun motif ne correspond, une erreur est levée. Nous allons
largement exploiter ceci.

# Erreurs de syntaxe

Le système de macro levera une erreur si aucun motif ne correspond
à la forme, mais il nous deviendra utile d'écrire des motifs qui
rejettent explicitement une forme si le motif correspond
*exactement*. Ceci est aisément réalisé en faisant en sorte que le
template associé à un motif s'expanse en code mal formé, mais le
message d'erreur résultant est assez peu utile :

     (define-syntax prohibit-one-arg
       (syntax-rules ()
         ((prohibit-one-arg function argument) (if)) ;; bogus expansion
         ((prohibit-one-arg function . arguments)
          (function . arguments))))
    
     (prohibit-one-arg + 2 3)
       => 5
    
     (prohibit-one-arg display 'foo)
      if: bad syntax (has 0 parts after keyword) in: (if)

Pour générer un message d'erreur plus utile, et pour indiquer à
l'intérieur de la définition de macro que l'expansion bugguée est
intentionnelle, nous allons définir une macro faite conçue pour
lever une erreur de syntaxe. (C'est sans le moinde doute une façon
procédurale de ce faire, mais nous souhaitons lever cette erreur
pendant l'expansion de la macro et le langage de motif ne fournit
pas de moyen de le faire directement. Nous pourrions utiliser une
macro plus compliquée, mais celle est belle, facile et portable.)

      (define-syntax syntax-error (syntax-rules () ((syntax-error)
      (syntax-error "Bad use of syntax error!"))))

Nous pouvons maintenant écrire des macros qui s'expansent en
'appels' à syntax error. Si l'appel contient plus d'un argument, le
motif ne correspondra pas et une erreur sera levée. La plupart des
systèmes de macro Scheme affichent la forme qui a échoué à
correspondre, de sorte que nous pouvons y mettre nos messages de
débug.

    (define-syntax prohibit-one-arg
       (syntax-rules ()
         ((prohibit-one-arg function argument)
          (syntax-error
           "Prohibit-one-arg cannot be used with one argument."
           function argument))
         ((prohibit-one-arg function . arguments)
          (function . arguments))))
    
    (prohibit-one-arg display 3)
    => syntax-error: bad syntax in: (syntax-error "Prohibit-one-arg cannot
    be used with one argument." display 3)


## Écrire un macro d'erreur de syntaxe.

Écrivez des motifs de 'rejet' qui s'expansent en appels de
syntax-error.

# Correspondance 'accidentelle'

Le motif ressemble a peu près à ce à quoi il correspondra, mais ce
peut être une source de confusion. Un motif tel que celui ci :
(my-named-let name (binding . more-bindings) . body)

Correspondra à cette forme : (my-named-let ((x 22) (y "computing
square root")) (display y) (display (sqrt x)))

comme ci-dessous : name = ((x 22) (y "computing square root"))

binding = display more-bindings = (y)

body = ((display (sqrt x)))

*Les structures de listes imbriquées dans le motif pourront correspondre à des structures de listes imbriquées de façon analogue, mais les symboles dans le motif correspondront à _n'importe quoi_.*

Dans cet exemple nous voulons interdire la correspondance de NAME
avec une liste. Nous le réalisons en 'défendant' le motif en
question avec des motifs qui ne devraient pas être permis :
    (define-syntax my-named-let
      (syntax-rules ()
    
        ((my-named-let () . ignore)
         (syntax-error "NAME must not be the empty list."))
    
        ((my-named-let (car . cdr) . ignore)
         (syntax-error "NAME must be a symbol." (car . cdr)))
    
        ((my-named-let name bindings form . forms) ;; begin implicite
         (let name bindings form . forms))))
    
*Eviter les correspondances de motifs accidentelles en écrivant des patterns de défense qui correspondent à une mauvaise syntaxe.*

# Expansion récursive

Notre macro syntax-error peut s'expanser en une forme syntax-error.
Si le résultat de l'expansion de la macro est lui même une forme
macro, cette macro résultante sera expansée. Ce traitement continue
jusqu'a ce que soit la forme résultante n'est pas un appel de
macro, soit la forme résultante ne peut pas correspondre à un
motif. La macro syntax-error est conçue pour ne pouvoir
correspondre qu'a un appel sans arguments. Un appel sans arguments
à syntax-error s'expand en un appel à syntax-error à un argument,
qui ne peut pas correspondre.

Le template d'une macro peut s'expandre en une forme qui embarque
en son sein un appel à la même macro. Le code embarqué code sera
expansé normalement si le code environnant le traite comme une
forme normale. L'appel embarqué contiendra normalement moins de
formes que l'appel original de sorte que l'expansion s'achèvera
avec une expansion tiviale.

Notez, cependant, que la macro récursive ne sera pas expansée à
moins que le code intermédiaire l'utilise aussi comme une macro.
C'est pourquoi the truc de débuggage qui consiste à quoter le
template fonctionne : la forme macro est expansée à une forme
quote. La forme quote ne fait que traiter ses arguments comme des
données et aucune expansion n'est effectuée.

Les expansions récursives produisent toujours un résultat imbriqué.
C'est utilisé à bon escient dans cet exemple :

     (define-syntax bind-variables
       (syntax-rules ()
         ((bind-variables () form . forms)
          (begin form . forms))
    
         ((bind-variables ((variable value0 value1 . more) . more-bindings) form . forms)
          (syntax-error "bind-variables illegal binding" (variable value0 value1 . more)))
    
         ((bind-variables ((variable value) . more-bindings) form . forms)
          (let ((variable value)) (bind-variables more-bindings form . forms)))
    
         ((bind-variables ((variable) . more-bindings) form . forms)
          (let ((variable #f)) (bind-variables more-bindings form . forms)))
    
         ((bind-variables (variable . more-bindings) form . forms)
          (let ((variable #f)) (bind-variables more-bindings form . forms)))
    
         ((bind-variables bindings form . forms)
          (syntax-error "Bindings must be a list." bindings))))

BIND-VARIABLES ressemble beaucoup à LET*, mais vous autorise à
omettre la valeur dans la liste des liens. Si vous oubliez cette
valeur, la variable sera liée à #f. Vous pouvez aussi omettre les
parenthèses autour du nom de la variable.
     (let ((a 1))
        (bind-variables ((b)
                         c
                         (d (+ a 3)))
          (list a b c d)))
    
Quand bind-variables est traité, l'expansion immédiate est ceci :
    (let ((a 1))
        (bind-variables ((b)
                         c
                         (d (+ a 3)))
          (list a b c d)))

La forme LET est maintenant traitée et, en tant que partie de ce
traitement, le corps du LET sera expansé.

     (bind-variables ((b)
                         c
                         (d (+ a 3)))
          (list a b c d))
    
Ceci s'expand dans une autre forme LET :

    (let ((a 1))
      (let ((b #f))
        (let ((c #f))
          (let ((d (+ a 3)))
            (list a b c d)))))
    
Le traitement se termine quand la liste des liens est la liste
vide. A ce moment, chaque niveau d'expansion est replacé dans son
contexte environnant pour retourner la structure imbriquée suivante: 

Il y a plusieurs choses à noter ici : 
1. la macro utilise l'idiome
   du begin implicite 
2. Il y a des motifs de 'défense' pour faire
   correspondre des liens avec plus d'une valeur de forme. 
3. Dans chaque expansion, le premier lien sera ôté. Les liens restants
   apparaissent en position de lien à l'appel récursif, ainsi, la
   macro est inductive sur la liste des liens; 
4. Il y a un cas de
   base de l'induction qui fait correspondre une liste de liens vide.
5. Sur chaque itération, on tente de faire correspondre le premier
   lien et ces motifs dans l'ordre : 
        (variable value0 value1 . more) 
   une liste de plus de deux éléments 
       (variable value)
   une liste
   d'exactement 2 éléments 
       (variable) 
   une liste d'un élément 
       variable
   pas une liste 

Cet ordre est utilisé parce que le dernier motif
correspondra à n'importe quoi et empêchera de faire correspondre
les correspondances précédentes.

Cet idiome de macro est aussi extrèmement commun.

## L'idiome de récursion de liste

Cet idiome est utilisé pour traiter récursivement une liste de
sousformes macro.

La macro a une liste parenthésée d'éléments en position fixe. La
liste peut être vide (), mais ne peut pas être omise.

Un motif qui fait correspondre la liste vide à cette position
précède les autres correspondances. Le template pour ce motif
n'inclut pas une autre utilisation de cette macro.

Il y a a un ou plusieurs motifs qui on une suite pointée en
position de liste. Les motifs sont ordonnés du plus spécifique au
plus général afin d'assurer que les correspondances les plus
tardives ne 'cachent' pas les plus précoces. Les templates associés
sont soit des syntax-errors, soit contiennent un autre usage de
cette macro. La position de la liste dans l'appel récursif
contiendra le composant de suite pointée.

Une récursion de liste minimal ressemble à ce qui suit :

     (define-syntax mli
       (syntax-rules ()
    
         ((mli ()) (base-case))
    
         ((mli (item . remaining)) (f item (mli remaining)))
    
         ((mli non-list) (syntax-error "not a list")))))
    
Notez que l'appel récursif dans la seconde clause utilise la
variable motif REMAINING et qu'elle n'est PAS pointée. Chaque appel
récursif contient par conséquent une liste plus courte de formes au
même point.

* * * * *

A ce point, les choses se compliquent. Nous ne pouvons plus
regarder les macros comme de 'simples réécritures'. Nous commençons
à écrire des macros dont le but est de contrôler les actions du
moteur de traitement de macros. Nous réécrirons des macros dont le
but n'est plus de produire du code mais d'effectuer un traitement.

Une macro est un compilateur. Il prend du code source écrit dans un
langage, c.a.d Scheme avec quelque extension syntaxique, et il
génère du code objet écrit dans un autre langage, c.a.d. Scheme
*sans* cette extension syntaxique. Le langage dans lequel nous
allons écrire ces compilateurs n'est PAS Scheme. C'est le langage
de motif et de templates de syntax-rules.

Un compilateur consiste en trois phases simples: un analyseur
syntaxique qui lit le langage source, un traitement de
transformation et de traduction qui fait correspondre la sémantique
du langage source en constructions du langage cible, et un
générateur qui transforme les constructions cibles résultantes en
code. Nos macros auront ces trois parties identifiables. La phase
d'analyse syntaxique utiliser le langage de motifs pour extraire
des fragments de code du code source. Le traducteur agira sur ces
fragments de code et les utilisera pour construire et manipuler des
fragments de code Scheme. La phase de génération assemblera les
fragments Scheme de code Scheme avec un template qui sera le
résultat final.

Le langage de motif et template de syntax-rules est un langage
d'implantation inhabituel. Le modèle de règle dirigé par les
templates rend la génération de code triviale. L'hygiène
automatique trace le contexte des fragments de code et,
principalement, nous permet de manipuler les fragments de code
immédiats comme des éléments d'un arbre syntaxique abstrait. Il n'y
a qu'un problème: le modèle de calcul est non-procédural. Les
abstractions de programmation standard telles que les
sous-routines, variables nommées, données structurées et
conditionnelles ne sont pas simplement différentes en Scheme, elles
n'existent pas dans une forme reconnaissable !

Mais si nous regardons avec attention, nous découvrirons que nos
abstractions de programmation familières n'ont pas disparues du
tout --- elles ont été déstructurées, découpées et reconstituées en
formes étranges. Nous allons écrire du code à l'aspect étrange. Ce
code sera écrit sous une forme de transformateur macro, mais ce ne
sera pas une macro dans le sens traditionnel.

Quand une macro est expansée, la forme originale est réécrite sous
une nouvelle forme. Si le résultat de la macro-expansion est une
nouvelle forme macro, l'expanseur expand alors ce résultat. Donc si
la macro s'expand en un autre appel à elle même, nous avons écrit
une boucle récursive terminale. Nous avons fait une utilisation
minimale de ceci au dessus avec la macro syntax-error, mais à
partir de maintenant nous feront de ceci une pratique standard.

Nous sommes tombés sur la récursion de liste plus haut. Nous
pouvons créer une itération de liste analogue :

     (define-syntax bind-variables1
       (syntax-rules ()
         ((bind-variables1 () form . forms)
          (begin form . forms))
    
         ((bind-variables1 ((variable value0 value1 . more) . more-bindings) form . forms)
          (syntax-error "bind-variables illegal binding" (variable value0 value1 . more)))
    
         ((bind-variables1 ((variable value) . more-bindings) form . forms)
          (bind-variables1 more-bindings (let ((variable value)) form . forms)))
    
         ((bind-variables1 ((variable) . more-bindings) form . forms)
          (bind-variables1 more-bindings (let ((variable value)) form . forms)))
    
         ((bind-variables1 (variable . more-bindings) form . forms)
          (bind-variables1 more-bindings (let ((variable #f)) form . forms)))
    
         ((bind-variables1 bindings form . forms)
          (syntax-error "Bindings must be a list." bindings))))

Parce que nous traitons en premier la variable la plus à gauche, la
forme résultante sera imbriquée dans l'ordre inverse de la version
récursive. Nous traiterons ce point par la suite et n'écrivons pour
l'instant que la liste des liens. 

    (bind-variables1 ((d (+ a 3)) ;; a is visible in this scope.
                      c            ;; c will be bound to #f
                      (b)          ;; so will b
                      (a 1))       ;; a will be 1
        (list a b c d))
    
La première macro s'expand en ceci : 
    (bind-variables1 (c
                      (b)
                      (a 1))
      (let ((d (+ a 3)))
        (list a b c d)))
    
Mais comme cette forme est une macro, l'expanseur est à nouveau
lancé. Cette seconde expansion résulte en ceci : 

    (bind-variables1 ((b)
                      (a 1))
      (let ((c #f))
        (let ((d (+ a 3)))
          (list a b c d))))

L'expanseur est à nouveau lancé pour produire ce qui suit :

    (bind-variables1 ((a 1))
      (let ((b #f))
        (let ((c #f))
          (let ((d (+ a 3)))
            (list a b c d)))))

Une autre itération produit ceci : 

    (bind-variables1 ()
      (let ((a 1))
        (let ((b #f))
          (let ((c #f))
            (let ((d (+ a 3)))
              (list a b c d))))))

La prochaine expansion ne contient pas d'appel à la macro : 

    (begin
      (let ((a 1))
        (let ((b #f))
          (let ((c #f))
            (let ((d (+ a 3)))
              (list a b c d))))))

Nous pourrons appeler ceci l'"Idiome d'Itération de Liste", mais
amenons ceci dans une direction toute différente.

Remarquez que le template utilisé pour la plupart des règles
commence avec le nom de la macro lui-même, afin de faire en sorte
que le macro-expanseur ré-expanse le résultat immédiatement. Mais
ignorons l'expanseur et faisons comme si
*le template est un appel de fonction récursif terminal*.

En supprimant la vérification d'erreurs et les formats d'arguments
multiples pour les liens d'arguments, nous pouvons voir le principe
de ce qui se passe:

    (define-syntax bind-variables1
       (syntax-rules ()
         ((bind-variables1 () . forms)
          (begin . forms))
    
         ((bind-variables1 ((variable value) . more-bindings) . forms)
          (bind-variables1 more-bindings (let ((variable value)) . forms)))))

BIND-VARIABLES1 est une fonction récursive terminale. SYNTAX-RULES
se comporte comme une expression COND. Les arguments de
bind-variables1 sont anonymes, mais en plaçant les motifs dans la
position appropriée, nous pouvons déstructurer les valeurs passées
dans les variables nommées.

Démontrons cette idée via un exemple simple.

Nous voulons reproduire le comportement de la macro Common Lisp
MULTIPLE-VALUE-SETQ. Cette fome prend comme argument une liste de
variables et de formes qui renvoient des valeurs multiples. La
forme est invoquée et les variables sont SET! À leurs valeurs de
retour respectives. Une expansion putative pourrait être :

     (multiple-value-set! (a b c) (generate-values))
      =>  (call-with-values (lambda () (generate-values))
            (lambda (temp-a temp-b temp-c)
              (set! a temp-a)
              (set! b temp-b)
              (set! c temp-c)))

Par souci de clarté, nous commencerons sans vérification d’erreurs.
Nos macros étant récursives terminales, nous pouvons écrire
plusieurs sous-routines pour chaque partie de l'expansion.

Premièrement, nous écrivons la macro d'"entrée. C'est cette
macro qui sera exportée à l'utilisateur. Cette macro appellera
celle qui créé les éléments temporaires et les opérations
nécessaires d'assignation. Elle passe des listes vides comme
valeurs initiales pour ces expressions.
    (define-syntax multiple-value-set!
      (syntax-rules ()
        ((multiple-value-set! variables values-form)
    
         (gen-temps-and-sets
             variables
             ()  ;; initial value of temps
             ()  ;; initial value of assignments
             values-form))))

En supposant que gen-temps-and-sets fait correctement le travail,
nous voudrons émettre le code résultant. Celui-ci est évident:

    (define-syntax emit-cwv-form
      (syntax-rules ()
    
        ((emit-cwv-form temps assignments values-form)
         (call-with-values (lambda () values-form)
           (lambda temps . assignments)))))

Les variables temporaires et les assignations sont simplement
collées dans aux bons endroits.

Maintenant, nous avons besoin d'écrire la routine qui génère une
variable temporaire et son assignation pour chaque variable. Nous
allons encore utiliser l'induction sur la liste des variables.
Quand cette liste est vide, nous appellerons EMIT-CMV-FORM avec les
résultats collectés. A chaque itération nous enlèverons une
variables, générerons la variable temporaire et l'assignation pour
cette variable, et nous les collecterons avec les autres variables
temporaires et les assignations.
    (define-syntax gen-temps-and-sets
      (syntax-rules ()
    
        ((gen-temps-and-sets () temps assignments values-form)
         (emit-cwv-form temps assignments values-form))
    
        ((gen-temps-and-sets (variable . more) temps assignments values-form)
         (gen-temps-and-sets
            more
           (temp . temps)
           ((set! variable temp) . assignments)
           values-form))))
     
* * * * *

Avant de développer plus avant, cependant, il y a quelques points
importants au sujet de cette macro qui ne doivent pas être
négligés.

> Vous ai-je déjà dit que Mme McCave Avait vingt-trois fils, et
> qu'elle les a tous nommés Dave ? – Dr. Seuss

Notre macro MULTIPLE-VALUE-SET! Génère une variable temporaire pour
chacune des variables qui sera assignée (ceci se fait dans la
seconde clause de GEN-TEMPS-AND-SETS). Malheureusement, toutes les
variables temporaires sont nommées TEMP. Nous pouvons le voir si
nous imprimons le code expansé :

    (call-with-values (lambda () (generate-values))
      (lambda (temp temp temp)
        (set! c temp)
        (set! b temp)
        (set! a temp)))

> Hé bien, c'est ce qu'elle fit. Et ça n'a pas été très malin. Vous
> voyez, quand elle veut en appeler un et qu'elle crie "Youhou, viens
> à la maison, Dave!", il n'en vient pas un. Tous les vingt-trois
> Daves accourent!" (Ibid.)

L'importance d'identifiant uniques afin d'éviter les collisions de
noms est appris à un très jeune age.

La chose amusante est que cependant le code fonctionne. Il y a en
fait trois objets syntaxiques différents (tous nommés temp) qui
seront liés par la forme LAMBDA, et chacun des SET! Fait référence
au bon. Mais il y a six identifiants avec le nom TEMP. Pourquoi
donc le macro-expanseur a t il décidé de créer trois paires
d'objets syntaxiques associés plutôt que six objets syntaxiques
individuels, ou encore deux triplets? La réponse réside dans ce
template:

    (gen-temps-and-sets
            more
           (temp . temps)
           ((set! variable temp) . assignments)
           values-form)

La variable TEMP qui est ajoutée dans la liste des variables
temporaires et la variable TEMP de la forme nouvellement créée
(SET! VARIABLE TEMP) feront référence au même objet syntaxique
parce qu'ils sont créés durant la même étape d'expansion. Ce sera
un objet syntaxique différent qui sera créé durant une autre étape
d'expansion.

Puisque nous lançons l'étape d'expansion trois fois, une pour
chaque variable à assigner, nous obtenons trois variables nommés
temp. Elles sont correctement appariées parce que nous avons généré
leurs références au même moment.

*Introduisez les fragments de code associés pendant la même étape d'expansion.*

*Introduisez les fragments dupliqués mais indépendants pendant des étapes d'expansion différentes.*

Explorons deux variantes de ce programme. Nous séparerons la
génération des variables temporaires et les assignements en deux
fonctions: GEN-TEMPS et GEN-SETS.

GEN-TEMPS et GEN-SETS iitérant sur la liste de variable au moment
de leur traitement, nous modifions MULTIPLE-VALUE-SET! Pour passer
la liste deux fois. GEN-TEMP fera l'induction sur une copie,
GEN-SETS travaillera avec l'autre.

    (define-syntax multiple-value-set!
      (syntax-rules ()
        ((multiple-value-set! variables values-form)
    
         (gen-temps
             variables ;; provided for GEN-TEMPS
             ()  ;; initial value of temps
             variables ;; provided for GEN-SETS
             values-form))))
     

GEN-TEMPS effectue l'induction sur la première des variables et
créé une liste des variables temporaires.

    (define-syntax gen-temps
      (syntax-rules ()
    
        ((gen-temps () temps vars-for-gen-set values-form)
         (gen-sets temps
                   vars-for-gen-set
                   () ;; initial set of assignments
                   values-form))
    
        ((gen-temps (variable . more) temps vars-for-gen-set values-form)
         (gen-temps
           more
           (temp . temps)
           vars-for-gen-set
           values-form))))


GEN-SETS effectue l'induction sur la liste de variables et créé la
liste des formes d'assignation.

    (define-syntax gen-sets
      (syntax-rules ()
    
        ((gen-sets temps () assignments values-form)
         (emit-cwv-form temps assignments values-form))
    
        ((gen-sets temps (variable . more) assignments values-form)
         (gen-sets
           temps
           more
           ((set! variable temp) . assignments)
           values-form))))

Bien que le résultat de l'expansion semble similaire, cette version
de la macro ne fonctionne pas. Le nom TEMP qui est généré dans
l'expansion GEN-TEMPS est différent du nom TEMP généré dans
l'expansion des GEN-SETS. L'expansion contiendra six identifiants
indépendants (tous nommés TEMP).

Mais nous pouvons réparer ceci. Remarquez que GEN-SETS transporte
la liste des variables temporaires générées par GEN-TEMPS. A la
place de générer une nouvelle variable TEMP, nous pouvons extraire
le TEMP précédemment généré de la liste et l'utiliser.

D'abord nous arrangeons GEN-TEMPS pour qu'il passe les temps deux
fois. Nous avons besoin de destructurer une des listes pour
effectuer l'itération, mais nous avons besoin de la liste complète
pour l'expansion finale.

    (define-syntax gen-temps
      (syntax-rules ()
    
        ((gen-temps () temps vars-for-gen-set values-form)
         (gen-sets temps
                   temps ;; another copy for gen-sets
                   vars-for-gen-set
                   () ;; initial set of assignments
                   values-form))
    
        ((gen-temps (variable . more) temps vars-for-gen-set values-form)
         (gen-temps
           more
           (temp . temps)
           vars-for-gen-set
           values-form))))



GEN-SETS utilise encore l'induction de liste, mais sur deux listes
parallèles plutot que si une liste unique (les listes ont la même
taille). A chaque boucle, nous lions TEMP à une variable temporaire
précédemment générée.

    (define-syntax gen-sets
      (syntax-rules ()
    
        ((gen-sets temps () () assignments values-form)
         (emit-cwv-form temps assignments values-form))
    
        ((gen-sets temps (temp . more-temps) (variable . more-vars) assignments values-form)
         (gen-sets
           temps
           more-temps
           more-vars
           ((set! variable temp) . assignments)
           values-form))))


A présent, nous ne générons plus de nouvelles variables temporaires
dans GEN-SETS, mais nous réutilisons les anciennes qui ont été
générées par GEN-SETS.

# Ellipses

Les ellipses ne sont pas réellement difficiles à comprendre, mais
elles ont trois sérieux problèmes qui les rendent mystérieuses
quand on tombe dessus pour la première fois. - L'opérateur … semble
très étrange. Ca n'est pas un élément de syntaxe «normal» et il a
déjà une signification en Français qui est assez bizarre compte
tenu de ce qu'il fait. Les macros qui utilisent les ellipses
ressemblent à des phrases d'accroches de critiques de films. -
L'élément de syntaxe … est un mot réservé dans le langage macro. A
la différence de tous les autres éléments de syntaxe, il ne peut
pas être re-lié ou quoté. Dans le langage macro, … agit comme une
marque syntaxique de même nature qu'une double quote, une suite
pointée ou une parenthèse. (Il est aberrant d'utiliser une
partenthèse ouvrante comme nom de variable! La même chose est vraie
pour … dans le langage macro.) Mais en Scheme standard, … est un
élément syntaxique d'identification et on peut concevoir de
l'utiliser comme variable. - De toutes les formes en Scheme,
l'opérateur … est le seul qui est POSTFIX! L'opérateur le mode
d’interprétation de la forme PRECEDENTE dans le langage macro. La
seule exception est la cause probable de la plupart de la
confusion. (Bien sur, si vous deviez utiliser … comme fonction dans
le Scheme normal, c'est un opérateur préfixe).

Dans un motif de macro, les ellipses ne peuvent apparaître que dans
le dernier élément d'un motif LIST ou VECTEUR. Il doit y avoir un
motif immédiatement avant (puisque c'est un opérateur préfixe), de
sorte que les ellipses ne peuvent apparaître après une parenthèse
ouvrante non fermée. Au sein d'un motif, les ellipses rendent le
motif les précédent capable de correspondre à des occurrences
répétées d'elles-mêmes à la fin de la liste ou du vecteur qui les
contient. En voici un exemple.

Supposons que notre motif est le suivant:

    (foo var #t ((a . b) c) ...)

Ce motif correspondrait à :

    (foo (* tax x) #t ((moe larry curly) stooges))

puisque var peut correspondre à tout, que \#t peut correspondre à
\#t et qu'une occurrence de ((a . b) c) peut correspondre à . a
correspondrait à moe, b correspondrait à la suite (larry curly) et
c correspondrait à stooges.

Ce motif correspondrait à :

     (foo 11 #t ((moe larry curly) stooges)
                 ((carthago delendum est) cato)
                 ((egad) (mild oath)))        

puisque var peut correspondre à tout, \#t à \#t et trois
occurrences du motif ((a . b) c) pourrait chacune correspondre aux
trois éléments restant. Remarquez que le troisième élément peut
correspondre à a pour egad, b étant vide et le motif c correspond à
la liste entière (mild oath).

Ce motif ne pourrait _pas_ correspondre à : 

      (foo 11 #t ((moe larry curly) stooges)
                     ((carthago delendum est) cato)
                     ((egad) mild oath))

La raison en est que l'argument final ((egad) mild oath) ne peut
pas correspondre à ((a . b) c) parce que le motif est une liste de
deux éléments et que nous avons fourni une liste de trois
éléments.

Ce motif correspondrait à :

    (foo #f #t)

parce que var peut correspondre à tout, \#t à \#t et zéro
occurrences du motif ((a . b) c) pourraient correspondre à la suite
vide)

*les ellipses doivent être en fin d'un motif liste ou vecteur. La liste ou le vecteur doit avoir au moins un motif initial.*

*Les ellipses sont un opérateur postfixe qui agit sur le motif les précédant.*

*Les Ellipses permettent à la liste ou au vecteur la contenant de pouvoir correspondre à un motif, du moment que l'élément de tête du motif correspond à l'élément de tête de la liste ou du vecteur jusqu'à (sans l'inclure) le motif avant l'ellipse, _et_ zero ou plus copies de ce motif peuvent correspondre _chacun et tous_ les éléments restants.*

*Souvenez vous que zéro occurrences du motif peuvent 'correspondre'.*

Quand un motif a une ellipse, la variable motif contenue dans la
clause et précédent l'éllipse opère différemment de la normale, il
doit être suffixé d'une ellipse ou il doit être contenu dans une
sous forme constituant un template qui est suffixé par une ellipse.
Dans le template, tout élément suffixé par une ellipse sera répété
autant de fois que les variables motifs imbriquée correspondent.

Supposons que notre motif est (foo var \#t ((a . b) c) ...) et mis
en correspondance avec

    (foo 11 #t ((moe larry curly) stooges)
                  ((carthago delendum est) cato)
                  ((egad) (mild oath)))

Ce sont des exemples de templates et les formes produites:

     ((a ...) (b ...) var (c ...))
    
    => ((moe carthago egad)
        ((larry curly) (delendum est) ())
        11
        (stooges cato (mild-oath)))
    
      ((a b c) ... var)
    
    =>  ((moe (larry curly) stooges)
         (carthago (delendum est) cato)
         (egad () (mild oath)) 11)
    
    ((c . b) ... a ...)
    
    =>  ((stooges larry curly)
         (cato delendum est)
         ((mild oath))
         moe carthago egad)
    
     (let ((c 'b) ...) (a 'x var c) ...)
    
    =>  (let ((stooges '(larry curly))
              (cato '(delendum est))
              ((mild oath) '()))
    
           (moe 'x 11 stooges)
           (carthago 'x 11 cato)
           (egad 'x 11 (mild-oath)))
    
Il peut exister plusieurs formes ellipses indépdendantes dans le
motif: Les différents sous-motifs n'ont pas besoin d'avoir la même
longueur pour que le motif entier puisse correspondre.

         (foo (pattern1 ...) ((pattern2) ...) ((pattern3) ...))

pourra correspondre à 
       (foo (a b) ((b) (c) (d) (e)) ())


Quand un template est répliqué, il est répliqué autant de fois que
la variable motif incluse a pu correspondre, de sorte que si vous
utilisez des variables issues de différents sous-motifs, les
sous-motifs doivent être de la même longueur. (La correspondance de
motifs fonctionnera si les sous motifs sont de taille différentes,
mis la transcription du template échouera si un sous motif utilise
deux longueurs différentes).

Vous pouvez être arrivés à la conclusion que les ellipses sont trop
compliquées à manipuler, et que dans le cas général elles sont très
complexes, mais il y a quelques motifs d'utilisation très courants
qui sont faciles à comprendre. Démontrons en un exemple:

Nous souhaitons écrire une macro TRACE-SUBFORMS qui réécrit une
expression données that given an expression donnée de sorte que de
l'information est affichée quand chaque sous-forme est évaluée.
Commençons par utiliser notre motif d'induction de liste.
(define-syntax trace-subforms (syntax-rules () ((trace-subforms
traced-form) (trace-expand traced-form ())))) ;; initialize the
traced element list

    (define-syntax trace-subforms
      (syntax-rules ()
        ((trace-subforms traced-form)
         (trace-expand traced-form
                       ())))) ;; intialise la liste des éléments tracés
    
    (define-syntax trace-expand
      (syntax-rules ()
    
        ;; sans plus de sous-formes, on emet le code tracé
        ((trace-expand () traced)
         (trace-emit . traced))
    
        ;; Autrement, on enrobe du code autour de la forme et on le cons à
        ;; la liste des formes tracées.
        ((trace-expand (form . more-forms) traced)
         (trace-expand more-forms
           ((begin (newline)
                   (display "Evaluating ")
                   (display 'form)
                   (flush-output)
                   (let ((result form))
                     (display " => ")
                     (display result)
                     (flush-output)
                     result))
             . traced)))))
    
    (define-syntax trace-emit
      (syntax-rules ()
        ((trace-emit function . arguments)
         (begin (newline)
                (let ((f function)
                      (a (list . arguments)))
                (newline)
                (display "Now applying function.")
                (flush-output)
                (apply f a))))))      
    
Essayons la:
    (trace-subforms (+ 2 3))
    
    Evaluating 3 => 3
    Evaluating 2 => 2
    Evaluating + => #<primitive:+>
    Now applying function.
    apply: expects type <procedure> as 1st argument,
        given: 3; other arguments were: (2 #<primitive:+>)
             (trace-subforms (+ 2 3))

Elle a évalué les sous-formes en sens inverse et a tenté d'utiliser
le dernier argument en tant que fonction. L'erreur est bien sur que
nous avons opéré de gauche à droite sur la liste des sous-formes,
lles résultats ont été ajoutés au début de la liste, de sorte
qu'ils apparaissent en sens inverse. Nous *pourrions* faire
intervenir une macro d'inversion de liste entre trace-expand et
trace-emit, mais nous pouvons utiliser les ellipses pour ce faire:

    (define-syntax trace-subforms
      (syntax-rules ()
    
        ((trace-subforms (form ...))
         (trace-emit (begin
                       (newline)
                       (display "Evaluating ")
                       (display 'form)
                       (flush-output)
                       (let ((result form))
                         (display " => ")
                         (display result)
                         (flush-output)
                         result)) ...))))
    
Le code effectuant le traçage sera répliqué autant de fois que FORM
peut correspondre, de sorte que les sous-formes seront passées à
trace-emit dans l'ordre correct. La sortie est maintenant:

    Evaluating + =\> \#<primitive:+\> 
    Evaluating 2 =\> 2 
    Evaluating 3 =\> 3 
    Now applying function.5

Vous avez peut-être remarqué que dans notre forme
MULTIPLE-VALUE-SET! Les assignations se déroulaient dans l'ordre
inverse de la liste de variables. Ca n'a pas vraiment d'importance
parce qu'effectuer une affectation n'affecte pas les autres. Mais
affichons quelque chose avant chaque affectation et faisons en
sorte que les affectations se déroulent dans l'ordre correct. Nous
allons réécrire notre MULTIPLE-VALUE-SET! En utilisant des
ellipses.

    (define-syntax multiple-value-set!
      (syntax-rules ()
        ((multiple-value-set! variables values-form)
    
         (gen-temps-and-sets
             variables
             values-form))))
    
    (define-syntax gen-temps-and-sets
      (syntax-rules ()
    
        ((gen-temps-and-sets (variable ...) values-form)
         (emit-cwv-form
             (temp)                          ;; ****
             ((begin
                (newline)
                (display "Now assigning ")
                (display 'variable)
                (display " to value ")
                (display temp)
                (force-output)
                (set! variable temp)) ...)
             values-form))))
    

Nous avons un problème. Nous avons besoin d'autant de variables
temporaires que de variables, mais la forme qui génère les
temporaires (marquée d'une astérisque) n'est pas répliquée parce
qu'elle ne contient pas de motif. Nous régleront ceci en placant la
variable à l'intérieur, avec les temporaires et en la décomposant
avec EMIT-CWV-FORM.
    
    define-syntax gen-temps-and-sets
      (syntax-rules ()
    
        ((gen-temps-and-sets (variable ...) values-form)
         (emit-cwv-form
             ((temp variable) ...)
             ((set! variable temp) ...)
             values-form))))
    
    (define-syntax emit-cwv-form
      (syntax-rules ()
    
        ((emit-cwv-form ((temp variable) ...) assignments values-form)
         (call-with-values (lambda () values-form)
           (lambda (temp ...)
               . assignments)))))
    
Nous avons un autre problème. Ca ne marche pas. Souvenez vous que
quand nous introduisons de nouvelles variables nous avons besoin
d'introduire des copies indépendantes dans des expansions
différentes. Même si nous créons une liste de temporaires, elles
sont toutes créées dans une seule expansion, de sorte qu'elles
auront toutes le même identifiant dans le code résutlant. Vous ne
pouvez pas utiliser le même identifiant deux fois dans une lambda
liste d'arguments.

Nous allons avoir besoin de retourner à la forme d'induction de
liste, mais le problème est qu'elle génère les affectations en
ordre inverse. Nous utiliserons donc les ellipses pour ne pas
itérer sur les variables, mais pour simuler APPEND:

    (define-syntax multiple-value-set!
      (syntax-rules ()
        ((multiple-value-set! variables values-form)
    
         (gen-temps-and-sets
             variables
             ()  ;; initial value of temps
             ()  ;; initial value of assignments
             values-form))))
    
    (define-syntax gen-temps-and-sets
      (syntax-rules ()
    
        ((gen-temps-and-sets () temps assignments values-form)
         (emit-cwv-form temps assignments values-form))
    
        ((gen-temps-and-sets (variable . more) (temps ...) (assignments ...) values-form)
         (gen-temps-and-sets
            more
           (temps ... temp)
           (assignments ... (begin
                              (newline)
                              (display "Now assigning value ")
                              (display temp)
                              (display " to variable ")
                              (display 'variable)
                              (flush-output)
                              (set! variable temp)))
           values-form))))
    
    (define-syntax emit-cwv-form
      (syntax-rules ()
    
        ((emit-cwv-form temps assignments values-form)
         (call-with-values (lambda () values-form)
           (lambda temps . assignments)))))
    
    (multiple-value-set! (a b c) (values 1 2 3))
    
     Now assigning value 1 to variable a
     Now assigning value 2 to variable b
     Now assigning value 3 to variable c

Utiliser le sous motif ellipse (temps …) dans le motif, avec le
sous template ellipse (temps … temp) dans le template, est analogue
à écrire (append temps (list temp)). C'est un idiome commun utilisé
pour étendre une list 'de droite à gauche'. Alors que concaténer du
coté droit est un idiome affreux quand utilisé en Scheme normal, il
est communément utilisé en expansion macro, parce que le temps
d'expansion est moins important que le temps d'exécution.

Nous pouvons maintenant voir comment "saucissonner" des sous listes
lors de la génération de code. Supposons un motif comme ceci:

     (foo (f1 ...) (f2 ...) . body-forms)

que nous ferions correspondre à ceci:

    (foo (a b c d) (1 2 3 4) moe larry curly)

Nous pouvons alors 'aplatir' la sortie en utilisant ce template:

     (f1... f2 ... . body-forms)

     =\> (a b c d 1 2 3 4 moe larry curly)

*Utilisez les ellipses pour étendre des listes tout en en
conservant l'ordre.*

*Utilisez des ellipses pour "aplatir" la sortie.*


* * * * *

Il n'est pas toujours commode d'écrire une macro auxiliaire séparée
pour chaque étape d'une macro compliquée. Il y a un truc qui peut
être utilisé pour placer des 'labels' sur des motifs. Comme une
chaine ne peut correspondre à une autre chaine que si elles sont
EQUAL?, nous pouvons placer une chaine dans nos motifs. Un template
qui souhaite simplement transférer à un motif particulier n'a qu'a
simplement mensionner la chaine. Nous pouvons combiner gen-sets,
gen-temps, et emit-cwv-forms avec ce truc:

    (define-syntax multiple-value-set!
      (syntax-rules ()
        ((multiple-value-set! (variable . variables) values-form)
         (mvs-aux "entry" variables values-form))
    
    (define-syntax mvs-aux
      (syntax-rules ()
        ((mvs-aux "entry" variables values-form)
         (mvs-aux "gen-code" variables () () values-form))
    
        ((mvs-aux "gen-code" () temps sets values-form)
         (mvs-aux "emit" temps sets values-form))
    
        ((mvs-aux "gen-code" (var . vars) (temps ...) (sets ...) values-form)
         (mvs-aux "gen-code" vars
                             (temps ... temp)
                             (sets ... (set! var temp))
                             values-form))
    
        ((mvs-aux "emit" temps sets values-form)
         (call-with-values (lambda () values-form)
           (lambda temps . sets)))))
    
* * * * *

Comparons cette forme à la macro GEN-TEMPS-AND-SETS

    (define-syntax gen-temps-and-sets
      (syntax-rules ()
    
        ((gen-temps-and-sets () temps assignments values-form)
         (emit-cwv-form temps assignments values-form))
    
        ((gen-temps-and-sets (variable . more) temps assignments values-form)
         (gen-temps-and-sets
            more
           (temp . temps)
           ((set! variable temp) . assignments)
           values-form))))        

avec du code Scheme qui itère sur une liste et construit une
collection de s-expressions. Ce code n'est pas une macro mais
effectue un traitement qui est superficiellement similaire.

    (define (gen-temps-and-sets variables temps assignments values-form)
       (cond
    
          ((null? variables)
           (emit-cwv-form temps assignments values-form))
    
          ((pair? variables) (let ((variable (car variables))
                                   (more     (cdr variables)))
    
                               (gen-temps-and-sets
                                  more
                                  `(temp ,@ temps)
                                  `((set! ,variable temp) ,@ assignments)
                                   values-form)))))


La similarité est frappante:

*   SYNTAX-RULES est un COND

*   Le langage de motifs est un hybride de prédicat (null? Et pair?) et de code de déstructuration de liste. Une paire pointée dans le motif prend le CAR et le CDR de la liste, du moment que c'est une PAIR? (.)

*   La structure de liste dans le template est QUASIQUOTE (sans ponctuation) et est équivalent aux formes CONS et LIST. Le quotage est implicite: tout est quoté excepté les identifiants qui ont correspondu dans le motif.

*   Les ellipses opèrent une sorte de mapcar/append.


Dans notre langage macro, nous avons défini les conditionnelles,
les variables, des appels récursifs terminaux, les structures de
données liste et paire et des primitives telles que CAR, CDR, CONS,
APPEND et même un mécanisme de correspondance bizarre. Que fait on
des appels de fonction no récursif terminaux, des abstractions de
structures de donneés, et des fonctions de première classe?
Malheureusement, ces constructions sont un peu plus complexes.

Le langage macro semble être une variante de Scheme, mais un peu
comme vu au travers du miroir ou les choses ne sont pas toujours ce
qu'elles semblent. Comment pouvons nous caractériser ce langage?

La différence la plus importante entre une forme en Schme et une
forme dans le langage macro est la signification des parenthèses.
En Scheme une forme parenthésée est soit une forme spéciale, soit
un appel de fonction. Dans le langage macro, c'est soit un prédicat
de déstructuration, soit un constructeur. Mais si nous utilisons
les parenthèses dans ce but, nous ne pouvons plus les utiliser pour
des appels de fonction. - un template dans le langage macro avec un
nom de macro à la place de la fonction est un appel de fonction
macro

*   comme en Scheme, la valeur de la variable est passée à la procédure appelée, mais non le nom de la variable.

*   comme en scheme, les arguments sont positionnels

*   à la différence de Scheme, les arguments ne sont pas nommés. Les motifs doivent être testés de façon positionnelle.

*   a la différence de Scheme, les sousexpressions dans le motifs ne peuvent invoquer des fonctions de manipulations de liste comme CAR, CDR, PAIR? Et EQUAL? Qu'indirectement.unlike Scheme, subexpressions in the pattern can only indirectly

*   a la différence de Schemen les sous expressions d'un template ne peuve qu'invoquer indirectement des fonctions de manipulation de liste telles que CONS ou APPEND via un mécanisme de quasiquotation.


Si nous n'avons pas la possibilité d'appeler des fonctions
quelconques depuis une sousexpression, nous sommes limités à une
séquence s'appel linéaire. Les macros vues jusqu'ici sont
suffisantes, mais nous aurons besoin de quelque chose de mieux pour
des macros plus compliquées.

Comment écrivons nous une sousroutine macro? Nous appels de macro
ont été jusqu'à maintenant récursifs terminaux --- chaque routine a
transféré le contrôle à une autre jusqu'à ce que le code émis soit
finalement appelé. Une sous routine, cependant a besoin d'une
continuation pour déterminer à quelle macro elle doit retourner.
Nous allons explicitement passer la continuation de la macro dans
la sousroutine macro. La sous routine 'retournera' en invoquant la
continuation de la macro sur les valeurs qu'elle a produites.

Utilisons cette technique pour inverser profondément une liste. La
'continuation' est représentée par une lite de chaines mot clés et
toute valeur dont la continuation peut avoir besoin (nous n'avons
pas encore les closures, donc nous trimbalons les données). Pour
retourner une valeur, nous déstructurons la continuation pour
obtenir le mot clé et ré-invoquer sreverse.
    
    (define-syntax sreverse
       (syntax-rules ()
         ((sreverse thing) (sreverse "top" thing ("done")))
    
         ((sreverse "top" () (tag . more))
          (sreverse tag () . more))
    
         ((sreverse "top" ((headcar . headcdr) . tail) kont)
          (sreverse "top" tail ("after-tail" (headcar . headcdr) kont)))
    
         ((sreverse "after-tail" new-tail head kont)
          (sreverse "top" head ("after-head" new-tail kont)))
    
         ((sreverse "after-head" () new-tail (tag . more))
          (sreverse tag new-tail . more))
    
         ((sreverse "after-head" new-head (new-tail ...) (tag . more))
          (sreverse tag (new-tail ... new-head) . more))
    
         ((sreverse "top" (head . tail) kont)
          (sreverse "top" tail ("after-tail2" head kont)))
    
         ((sreverse "after-tail2" () head (tag . more))
          (sreverse tag (head) . more))
    
         ((sreverse "after-tail2" (new-tail ...) head (tag . more))
          (sreverse tag (new-tail ... head) . more))
    
         ((sreverse "done" value)
          'value)))
      
Examinons précisément un simple appel et son retour. 
Cette clause
invoque récursivement sreverse sur le premier élément de
l'expression. Elle crée une nouvelle continuation en créant la
liste ("after-head" new-tail kont).  

    ((sreverse "after-tail" new-tail head kont)
          (sreverse "top" head ("after-head" new-tail kont)))
    

C'est là que nous finisssons quand la continuation est invoquée.
Nous nous attendons à ce que la "valeur de retour" soit le "premier
argument" suivant le tag. Les arguments restants après le taf sont
les éléments restant de la liste de la continuation que nous avons
faite. Comme vous pouvez le voir ici, "after-head" va elle-même
invoquer une continuation --- la continuation sauvegardée à
l'intérieur de la continuation construite au dessus.
    
    ((sreverse "after-head" () new-tail (tag . more))
          (sreverse tag new-tail . more))
    
         ((sreverse "after-head" new-head (new-tail ...) (tag . more))
          (sreverse tag (new-tail ... new-head) . more))

Il se passe quelque chose de très intéressant ici. Ajoutons une
règle à notre macro qui va faire en sorte de la macro s'arrête
quand la liste que nous souhaitons inverser est l'élément 'HALT.
    
    (define-syntax sreverse
       (syntax-rules (halt)
         ((sreverse thing) (sreverse "top" thing ("done")))
    
         ((sreverse "top" () (tag . more))
          (sreverse tag () . more))
    
         ((sreverse "top" ((headcar . headcdr) . tail) kont)
          (sreverse "top" tail ("after-tail" (headcar . headcdr) kont)))
    
         ((sreverse "after-tail" new-tail head kont)
          (sreverse "top" head ("after-head" new-tail kont)))
    
         ((sreverse "after-head" () new-tail (tag . more))
          (sreverse tag new-tail . more))
    
         ((sreverse "after-head" new-head (new-tail ...) (tag . more))
          (sreverse tag (new-tail ... new-head) . more))
    
         ((sreverse "top" (halt . tail) kont)
          '(sreverse "top" (halt . tail) kont))
    
         ((sreverse "top" (head . tail) kont)
          (sreverse "top" tail ("after-tail2" head kont)))
    
         ((sreverse "after-tail2" () head (tag . more))
          (sreverse tag (head) . more))
    
         ((sreverse "after-tail2" (new-tail ...) head (tag . more))
          (sreverse tag (new-tail ... head) . more))
    
         ((sreverse "done" value)
          'value)))
    
    (sreverse (1 (2 3) (4 (halt)) 6 7))
    =>
      (sreverse "top" (halt)
        ("after-head" ()
          ("after-tail2" 4
            ("after-head" (7 6)
              ("after-tail" (2 3)
                ("after-tail2" 1
                  ("done")))))))

Remarquez que chaque continuation de sreverse contient l'état
sauvegardé pour cette continuation, en plus de sa continuation
"mère"

Faisons un gros effort d'imagination:

La forme intermédiaire est une pile LIFO. L'élément du haut de la
pile est la fonction courante, les quelques éléments suivants sont
les arguments, suivis par l'adresse de retour et le contexte
dynamique restatn. Nous avons une pile de d'appels.

J'aime appeler ceci un "style de machine à pile".

Nous pouvons utiliser l'analogie entre le processeur macro et une
machine à pile plus complètement en réarrangeant les motifs et les
templates pour placer la continuation à un endroit défini.
L'endroit logique de la première continuation plutôt que de la
dernière. De plus, si nous modifions le mode de construction de la
continuation, c'est à dire que nous changeons le format du cadre de
la pile, nous pouvons faire en sorte que la valeur de retour soit
livrée à l'endroit que nous voulons. Plutôt que de placer la valeur
de retour immédiatement après le tag, nous placerons le tag et les
premières valeurs de du cadre de pile dans une liste et lors du
retour nous étalerons la liste et placerons la valeur après le
dernier élément. Ceci peut être fait à l'aide d'une macro
auxiliaire :
    
    (define-syntax return
      (syntax-rules ()
    
        ;; LA continuation vient en premier. La localisation de la valeur de retour est indiquée
        ;; à la fin de la liste
        ((return ((kbefore ...) . kafter) value)
         (kbefore ... value . kafter))
    
        ;; Le cas spécial qui ne fait que retourner la valeur de la continuation nulle.
        ((return () value) value)))
               (define-syntax return (syntax-rules ()
    
Construire une continuation de forme correcte est fastidieux, si
nous traitons quelque chose de la forme (f a b (g x) c d), nous
devons l'écrire de cette façon : (g ((f a b) c d) x) afin de
calculer G et obtenir le résultat placé dans (f a b c d). Nous
pouvons écrire une macro pour ça. Elle prendra trois arguments, la
continuation courante, les éléments sauvegardés qui viendront après
la valeur de retour, l'appel à la sous-routine et les éléments
sauvegardé après la valeur de retour. De sorte que si nous voulons
exprimer une fonction imbriquée comme celle ci: (f a b (g x) c d)
nous pouvons écrire

     (macro-subproblem (f a b) (g x) c d)

Nous sommes toujours limités à un seul sous-problème cependant.

Afin de faciliter les appels avec des tags, nous permettons que G
soit une liste qui est spread à son premier élément, ce qui fait
que

        (macro-subproblem (f a b) ((g "tag") x) c d)

devient
    (g "tag" ((f a b) (c d)) x)

    (define-syntax macro-subproblem
      (syntax-rules ()
        ((macro-subproblem before ((macro-function ...) . args) . after)
         (macro-function ... (before . after) . args))
    
        ((macro-subproblem before (macro-function . args) . after)
         (macro-function (before . after) . args))))
    
SREVERSE devient alors :

    (define-syntax sreverse
       (syntax-rules (halt)
         ((sreverse thing)
            (macro-subproblem
              (sreverse "done" ()) ((sreverse "top") thing)))
    
         ((sreverse "top" k ())
          (return k ()))
    
         ((sreverse "top" k ((headcar . headcdr) . tail))
          (macro-subproblem
            (sreverse "after-tail" k) ((sreverse "top") tail) (headcar . headcdr)))
    
         ((sreverse "after-tail" k new-tail head)
          (macro-subproblem
            (sreverse "after-head" k) ((sreverse "top") head) new-tail))
    
         ((sreverse "after-head" k () new-tail)
          (return k new-tail))
    
         ((sreverse "after-head" k new-head (new-tail ...))
          (return k (new-tail ... new-head)))
    
         ((sreverse "top" k (halt . tail))
          '(sreverse "top" k (halt . tail)))
    
         ((sreverse "top" k (head . tail))
          (macro-subproblem
            (sreverse "after-tail2" k) ((sreverse "top") tail) head))
    
         ((sreverse "after-tail2" k () head)
          (return k (head)))
    
         ((sreverse "after-tail2" k (new-tail ...) head)
          (return k (new-tail ... head)))
    
         ((sreverse "done" k value)
          (return k 'value))))
    

Bien que la syntaxe appelante est toujours assez pénible, nous
avons rendu facile l'écriture de sous-routines macro. Voici NULL?,
CAR, PAIR?, et LIST? écrits comme sous-routines macro:
    (define-syntax macro-null?
      (syntax-rules ()
        ((macro-null? k ())        (return k #t))
        ((macro-null? k otherwise) (return k #f))))
    
    (define-syntax macro-car
      (syntax-rules ()
        ((macro-car k (car . cdr)) (return k car))
        ((macro-car k otherwise)   (syntax-error "Not a list"))))
    
    (define-syntax macro-pair?
      (syntax-rules ()
        ((macro-car k (a . b))   (return k #t))
        ((macro-car k otherwise) (return k #f))))
    
    (define-syntax macro-list?
      (syntax-rules ()
        ((macro-car k (elements ...)) (return k #t))
        ((macro-car k otherwise)      (return k #f))))
    
La macro analogue à IF est explicite. Nous effectuons simplement
un appel enveloppé à pred après avoir inséré IF-DECIDE dans la
chaine de continuation.

    (define-syntax macro-if
      (syntax-rules ()
        ((macro-if (pred . args) if-true if-false)
         (pred ((if-decide) if-true if-false) . args))))
    
    (define-syntax if-decide
      (syntax-rules ()
        ((if-decide #t if-true if-false) if-true)
        ((if-decide #f if-true if-false) if-false)))

Un exemple en serait :

    (define-syntax whatis
      (syntax-rules ()
        ((whatis object) (whatis1 () object))))
    
    (define-syntax whatis1
      (syntax-rules ()
        ((whatis1 k object)
         (macro-if (macro-null? object)
           (return k 'null)
           (macro-if (macro-list? object)
             (macro-subproblem
               (whatis1 "after car" k proper) (macro-car object))
             (macro-if (macro-pair? object)
               (macro-subproblem
                 (whatis1 "after car" k improper) (macro-car object))
               (return k 'something-else)))))
    
        ((whatis1 "after car" k type c)
         (return k '(a type list whose car is c)))))


Le premier défaut évident à ces macros écrites en style de machine
à pile est la séquence d'appel maladroite. Nous ne pouvons traiter
qu'un sous-problème à la fois et nous avons besoin d'un label
auquel revenir pour chaque sous-problème. Comme vous pouvez
l'imaginer, ceci peut aussi être résolu par une fonciton macro.

Cependant, ceci est quelque peu problématique. Si nous adoptons
l'approche évidente et inspectons l'appel de macro à la recherche
d'expressions parenthésées, nous perdons notre capacité à utiliser
les templates comme templates. Nous aurons besoin d'être capables
de déterminer quelles parentèses ne sont pas utilisées comme appels
de fonction mais comme templates de listes. Nous pourrions
réintroduire une syntaxe de quotage, mais nous perdrions la plupart
de la puissance des templates et nous enterrerions notre code sous
une pile de formes de quote et unquote.

A la place, remplaçons les marques à l'intérieur des templates pour
indiquer que les sous expressions représentent en fait des
sous-problèmes.

    (define-syntax macro-call
      (syntax-rules (!)
        ((macro-call k (! ((function ...) . arguments)))
         (function ... k . arguments))
    
        ((macro-call k (! (function . arguments)))
         (macro-call ((macro-apply k function)) arguments))
    
        ((macro-call k (a . b))
         (macro-call ((macro-descend-right k) b) a))
    
        ((macro-call k whatever) (return k whatever))))
    
    (define-syntax macro-apply
      (syntax-rules ()
        ((macro-apply k function arguments)
         (function k . arguments))))
    
    (define-syntax macro-descend-right
      (syntax-rules ()
        ((macro-descend-right k evaled b)
         (macro-call ((macro-cons k evaled)) b))))
    
    (define-syntax macro-cons
       (syntax-rules ()
         ((macro-cons k ca cd) (return k (ca . cd)))))
        
Donc si nous expansons cette forme:
    
    (macro-call ()
      (this is a (! (macro-cdr (a b (! (macro-null? ())) c d))) test))

L'expansion est 
            (this is a (b \#t c d) test)

Ecrivons une macro similaire au LET. Les liens de type let étant
une fonctionnalité commune des macros, nous voudrions avoir une
sous-routine pour en faire l'analyse syntaxique. Nous aimerions
bien aussi une sous-routine pour nous dire si le premier argument
de notre LET est un nom ou une liste de liens.

    (define-syntax my-let
      (syntax-rules ()
        ((my-let first-subform . other-subforms)
         (macro-if (is-symbol? first-subform)
           (expand-named-let () first-subform other-subforms)
           (expand-standard-let () first-subform other-subforms)))))
    

Pour tester que quelque chose est un symbole, nous testons d'abord
afin de savoir s'il s'agit d'une liste ou d'un vecteur. Si il y a
une correspondance, nous savons que l'expression est atomique, mais
nous ne savons pas s'il s'agit d'un symbole. Souvenez-vous que les
symboles dans les macros peuvent être mis en correspondance avec
n'importe quoi, mais les autres atomes ne peuvent correspondre qu'a
ce qui leur est equal?. Oleg Kiselyov suggère cette méchante
bidouille: nous construisons une syntax-rule qui l'utilise et nous
regardons si la règle correspond à une liste.

    (define-syntax is-symbol?
      (syntax-rules ()
    
        ((is-symbol? k (form ...))  (return k #f))
        ((is-symbol? k #(form ...)) (return k #t))
    
        ((is-symbol? k atom)
         (letrec-syntax ((test-yes (syntax-rules () ((test-yes) (return k #t))))
                         (test-no  (syntax-rules () ((test-no)  (return k #f))))
                         (test-rule
                           (syntax-rules ()
                             ((test-rule atom) (test-yes))
                             ((test-rule . whatever) (test-no)))))
                (test-rule (#f))))))


Nous avons besoin d'utiliser LECTREC-SYNTAX ici parce que nous ne
voulons pas que nos formes de continuations soient dans le template
de test-rule; si la règle de test correspond, l'atome que nous
testons serait réécrit !

Faire l'analyse syntaxique d'une liste de liste de liens est facile
à faire avec une ellipse :

    (define-syntax parse-bindings
      (syntax-rules ()
        ((parse-bindings k ((name value) ...))
         (return k ((name ...) (value ...))))))
    
    (define-syntax expand-named-let
      (syntax-rules ()
        ((expand-named-let k name (bindings . body))
          (macro-call k
            (! (emit-named-let name (! (parse-bindings bindings)) body))))))
    
    (define-syntax expand-standard-let
      (syntax-rules ()
        ((expand-standard-let k bindings body)
         (macro-call k
            (! (emit-standard-let (! (parse-bindings bindings)) body))))))
    
    (define-syntax emit-named-let
      (syntax-rules ()
        ((emit-named-let k name (args values) body)
         (return k (((lambda (name)
                       (set! name (lambda args . body))
                        name) #f) . values)))))
      
La bidouille nécessaire.

Ce qui suit est un petit interpreteur scheme écrit dans une macro
syntax-rules. Il est incroyablement lent.
    
    (define-syntax macro-cdr
      (syntax-rules ()
        ((macro-cdr k (car . cdr)) (return k cdr))
        ((macro-cdr k otherwise)   (syntax-error "Not a list"))))
    
    (define-syntax macro-append
       (syntax-rules ()
         ((macro-append k (e1 ...) (e2 ...)) (return k (e1 ... e2 ...)))))
    
    (define-syntax initial-environment
      (syntax-rules ()
        ((initial-environment k)
         (return k ((cdr . macro-cdr)
                    (null? . macro-null?)
                    (append . macro-append)
                    )))))
    
    (define-syntax scheme-eval
      (syntax-rules ()
        ((scheme-eval expression)
         (macro-call ((quote))
           (! (meval expression (! (initial-environment))))))))
    
    (define-syntax meval
      (syntax-rules (if lambda quote)
        ((meval k () env)  (return k ()))
    
        ((meval k (if pred cons alt) env)
         (macro-if (meval pred env)
           (meval k cons env)
           (meval k alt env)))
    
        ((meval k (lambda names body) env)
         (return k (closure names body env)))
    
        ((meval k (quote object) env) (return k object))
    
        ((meval k (operator . operands) env)
         (meval-list ((mapply k)) () (operator . operands) env))
    
        ((meval k whatever env)
         (macro-if (is-symbol? whatever)
           (mlookup k whatever env)
           (return k whatever)))))
    
    (define-syntax mlookup
      (syntax-rules ()
        ((mlookup k symbol ())
         '(variable not found))
        ((mlookup k symbol ((var . val) . env))
         (macro-if (is-eqv? symbol var)
           (return k val)
           (mlookup k symbol env)))))
    
    (define-syntax meval-list
      (syntax-rules ()
        ((meval-list k evaled () env)
         (return k evaled))
    
        ((meval-list k (evaled ...) (to-eval . more) env)
         (macro-call k
           (! (meval-list (evaled ... (! (meval to-eval env))) more env))))))
    
    (define-syntax mapply
      (syntax-rules (closure)
        ((mapply k ((closure names body env) . operands))
         (macro-call k
           (! (meval body (! (extend-environment env names operands))))))
    
        ((mapply k (operator . operands))
         (macro-if (is-symbol? operator)
           (operator k . operands)
           '(non-symbol application)))))
    
    (define-syntax extend-environment
      (syntax-rules ()
        ((extend-environment k env () ())
         (return k env))
    
       ((extend-environment k env names ())
        '(too-few-arguments))
    
       ((extend-environment k env () args)
        '(too-many-arguments))
    
       ((extend-environment k env (name . names) (arg . args))
        (extend-environment k ((name . arg) . env) names args))))
    
