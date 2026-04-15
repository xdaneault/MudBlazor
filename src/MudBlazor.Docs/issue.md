# Description des issues effectuées

## 1re issue (PR : #12668, Issue : #11246) (merged)

### Description

Cette première issue consistait à régler un bug qui lançait une exception non gérée lorsqu'on initialisait le composant `MudDateRangePicker` avec `DateTime.MinValue`.

Cette valeur représente la plus petite valeur possible de la structure `DateTime` (00:00:00.0000000 UTC, January 1, 0001).

### Solution

Dans la classe `MudDateRangePicker.razor.cs`, qui est la classe du composant elle-même, j’ai dû créer deux nouvelles méthodes : `NormalizeDate` et `NormalizeDateRange`.

- `NormalizeDate` sert à normaliser une date précise.
  Elle vérifie si la date reçue en paramètre est `null` ou `DateTime.MinValue`.
  Si c’est le cas, elle retourne `null`, sinon elle retourne la date normalisée.
- `NormalizeDateRange` prend une plage de dates en paramètre.
  Le début et la fin de cette plage sont ensuite individuellement passés dans la méthode `NormalizeDate`.
  Si `DateTime.MinValue` est détecté, la valeur est convertie à `null`.
  La plage normalisée est ensuite retournée.

Finalement, avant toute autre action dans `SetDateRangeAsync`, je passe la plage de dates présente dans le champ du composant dans `NormalizeDateRange` afin d’intercepter l’exception avant qu’elle ne soit affichée à l’utilisateur.

### Tests ajoutés

En plus de régler ce problème, j’ai dû fournir trois tests unitaires présents dans la classe `DateRangePickerTest` :

- `InitializeDateRange_WithMinValue_ShouldTreatAsNull`
  Vérifie le cas où le composant `MudDateRangePicker` est initialisé avec deux dates `DateTime.MinValue` et s’assure que celles-ci sont normalisées à `null` pour éviter l’exception.
- `InitializeDateRange_WithMinValueStartOnly_ShouldNormalizeStart`
  Vérifie le cas où seulement la date de début est initialisée à `DateTime.MinValue` et que la date de fin est valide.  
  Le test s’assure que seule la date de début est normalisée à `null` et que la date de fin demeure inchangée.
- `InitializeDateRange_WithMinValueEndOnly_ShouldNormalizeEnd`
  Vérifie le cas inverse du test précédent, donc lorsque la date de début est valide et que la date de fin est `DateTime.MinValue`.

### Difficultés rencontrées

J’ai rencontré plusieurs problèmes lors de la résolution de cette issue.

Premièrement, j’ai passé beaucoup de temps à comprendre le flow du composant et l’ordre dans lequel les méthodes étaient appelées lors de l’initialisation et de l’utilisation du composant, afin de déterminer à quel endroit appliquer ma méthode de normalisation.

Deuxièmement, j’ai eu de la difficulté à comprendre la structure des tests, car ils utilisent le framework NUnit, que je n’avais jamais utilisé auparavant. De plus, ils utilisent de nombreux composants simulés à l’intérieur même des classes de test.

Au début, j’utilisais directement l’instance du picker, mais j’ai réalisé plus tard que je devais utiliser le composant interne du test pour que ma normalisation s’applique correctement.

J’ai donc perdu beaucoup de temps à me demander si le problème venait de ma solution ou de mes tests.

J’ai aussi eu des problèmes avec les pipelines automatiques qui échouaient sur GitHub :

- La première fois, c’était parce que je n’avais pas exécuté `dotnet format` afin de respecter les règles d’indentation du projet.
  Étant donné que je n'avais jamais utilisé cette commande, j'ai eu plusieurs erreurs et j'ai dû télécharger .NET 10.0, car les maintainers voulait absolument cette version d'indentations.
- La deuxième fois, c’était parce que j’avais directement ajouté un nouveau paramètre à `SetDateAsync`, ce qui est interdit, car cela demande une recompilation complète.

## 2e issue (PR : #12695, Issue : #12336) (merged)

### Description

Cette deuxième issue consistait à régler un bug visuel qui laissait une erreur de conversion affichée à l’écran lorsqu’un champ avec `Editable="true"` du composant `MudDatePicker` était rapidement effacé manuellement.

Il était seulement possible de supprimer l’erreur en remettant une date valide ou en effaçant le champ plus lentement.

Le problème provenait du mécanisme de debounce dans la méthode `SetDateAsync`. Cette méthode empêche certaines mises à jour lorsque la valeur précédente (`_value`) et la nouvelle valeur (`date`) sont identiques.

Lorsque l’utilisateur effaçait rapidement le champ, `_value` ainsi que date devenaient `null`, ce qui activait le guard et bloquait l’exécution de `BeginValidateAsync`. L’erreur de conversion restait donc affichée à l’écran.

### Solution

Pour régler ce problème, j’ai ajouté un paramètre optionnel `forceUpdate` (valeur par défaut : `false`) à la méthode `SetDateAsync`. Ce paramètre permet de contourner le debounce dans des cas précis.

Ensuite, dans la méthode `StringValueChangedAsync`, j’ai ajouté une condition qui vérifie si :

- le champ est vide ;
- une erreur de conversion était active auparavant.

Dans ce cas précis, j’appelle `SetDateAsync` avec `forceUpdate = true` afin de forcer la réexécution de la validation sans passer par le debounce.

Cela permet à `BeginValidateAsync` de s’exécuter et d’effacer l’erreur.

### Test ajouté

J’ai ajouté un test dans la classe `DatePickerTest` nommé :

`DatePicker_ShouldClearValidationError_WhenInvalidDateIsQuicklyErased`

Ce test reproduit exactement le scénario problématique :

1. Entrer une date invalide pour déclencher l’erreur de conversion.
2. Effacer rapidement le champ.
3. Vérifier que l’erreur disparaît correctement lorsque le champ devient vide.

### Difficultés rencontrées

Encore une fois, j’ai dû m’habituer à la façon dont les différents éléments de la classe et du composant interagissaient entre eux, particulièrement le mécanisme de debounce et la gestion interne de `_value`, `date`, ainsi que le cycle de validation du composant.

J’ai également eu des problèmes avec les pipelines automatiques qui échouaient sur GitHub :

- La première fois, c’était parce que je n’avais pas exécuté `dotnet format` afin de respecter les règles d’indentation du projet.
- La deuxième fois, c’était parce que j’avais directement ajouté un nouveau paramètre à `SetDateAsync`, ce qui est interdit, car cela demande une recompilation complète.

J’ai donc appliqué la solution proposée par les maintainers, qui consistait à utiliser deux overloads plutôt que de modifier directement la signature existante.

## 3e issue (PR : #12727, Issue : #10291) (merged)

### Description

Cette troisième issue consistait à régler un autre bug visuel dans le composant `MudList` lorsque le `SelectionMode` était configuré en `MultiSelection`.

Lorsqu’un élément était désactivé (`Disabled="true"`) mais faisait partie de la collection `SelectedValues` pendant l’initialisation, il n’affichait pas son état sélectionné visuellement (absence de checkmark).

Le problème était uniquement visuel et non logique : l’élément désactivé était bien présent dans `SelectedValues`, mais l’interface ne reflétait pas cet état.

Le problème provenait de la méthode `SetSelected`. Cette méthode retournait immédiatement si `GetDisabled` était `true`.

Ainsi, la variable `_selected`, responsable de l’état visuel, n’était jamais mise à jour pour les éléments désactivés.

### Solution

Pour corriger ce comportement, j’ai retiré `GetDisabled` de la condition `if` dans la méthode `SetSelected`.

Il est important de noter qu’après ce correctif, il est toujours impossible d’interagir avec des éléments désactivés. Le changement n’affecte que l’affichage visuel du checkbox lors de l’initialisation.

### Test ajouté

J’ai ajouté un test dans la classe `ListTest` nommé :

`ListMultiSelection_DisabledItems_ShouldDisplaySelectedStateAndNotBeClickable`

Ce test vérifie que :

- lorsque `MudList` est initialisé avec des éléments désactivés déjà présents dans `SelectedValues`, ceux-ci affichent correctement leur état sélectionné ;
- il demeure impossible de modifier leur état.

Afin de reproduire fidèlement le comportement dans l’environnement de test, j’ai ajouté des éléments avec `Disabled="true"` dans `ListMultiSelectionTest.razor`.

### Difficultés rencontrées

Je n’ai pas rencontré de difficultés majeures pour cette issue, car le correctif ne consistait qu’en une seule ligne et le composant est beaucoup moins complexe que les précédents.

J’ai toutefois dû retravailler ma PR à la suite de commentaires des maintainers :

- Une typo s’était glissée dans mon correctif.
- J’ai appliqué certaines suggestions proposées par Copilot et validées par les maintainers.

## 4e issue (PR : en cours, Issue : #12171)

### Description

Cette quatrième issue consistait à régler un bug de validation dans le composant `MudSelect`. Lorsqu’un `MudSelect` avec `Required="true"` était placé dans un `MudForm`, quitter le champ sans sélectionner de valeur n’affichait pas la bordure rouge ni le message d’erreur, contrairement au composant `MudTextField` qui se comportait correctement dans le même scénario.

La logique de validation fonctionnait tout de même en arrière-plan : la propriété `IsValid` du formulaire se mettait bien à jour à `false`, mais l’interface ne reflétait pas cet état.

### Solution

Après investigation, j’ai découvert que le problème venait de deux causes imbriquées.

Premièrement, le `MudInput` rendu à l’intérieur du `MudSelect` est configuré avec `ReadOnly="true"`. Or, la méthode `OnBlurredAsync` de `MudBaseInput` contient un guard `if (ReadOnly) return;` qui fait en sorte qu’elle retourne immédiatement sans jamais déclencher la validation ni invoquer le callback `OnBlur` du `MudSelect`.

Deuxièmement, l’événement qui capture réellement la perte de focus du composant est `@onfocusout` sur le div extérieur, géré par la méthode `OnFocusOutAsync`. Cette méthode ne traitait que le cas où le menu déroulant était ouvert (en redonnant le focus pour conserver la navigation au clavier dans le cas du multi-sélect), et ne faisait rien lorsque le menu était fermé.

Le correctif consiste donc à appeler `OnBlurredAsync` dans `OnFocusOutAsync` lorsque le menu est fermé, ce qui met correctement `Touched` à `true` et déclenche la validation sur l’instance de `MudSelect`.

À la suite des suggestions de Copilot validées par les maintainers, deux ajustements ont été apportés :

- La signature de `OnFocusOutAsync` a été modifiée pour accepter un paramètre `FocusEventArgs args`, provenant directement de la directive `@onfocusout`. Ce paramètre est ensuite transmis à `OnBlurredAsync(args)` au lieu d’instancier un `new FocusEventArgs()` vide, ce qui préserve les données réelles de l’événement (notamment la propriété `Type`).
- Dans le test, le sélecteur générique `"div.mud-select"` a été remplacé par `$"#{select.ElementId}"` afin de cibler précisément l’élément via son `id`, rendant le test plus robuste aux changements de markup futurs.

### Test ajouté

J’ai ajouté un test dans la classe `SelectTests` nommé :

`Select_Required_Should_ShowValidationError_OnFocusOut`

Ce test déclenche l’événement `onfocusout` sur le div extérieur du composant (ciblé par `ElementId`) et vérifie que :

- `Touched` passe à `true` ;
- `HasErrors` passe à `true` ;
- `ValidationErrors` contient le message `"Required"`.

### Difficultés rencontrées

Cette issue m’a demandé une investigation approfondie, car le bug n’était pas là où je l’attendais.

Ma première tentative consistait à modifier la méthode `OnBlurAsync` pour qu’elle appelle `OnBlurredAsync` au lieu d’invoquer directement le callback utilisateur. Cette modification faisait passer le test unitaire, mais n’avait aucun effet visuel, car `OnBlurAsync` n’est en réalité jamais appelée dans un scénario réel : le `ReadOnly="true"` sur le `MudInput` empêche le flow d’y arriver.

J’ai donc dû remonter la chaîne d’événements pour comprendre quel mécanisme capturait réellement la perte de focus, et c’est là que j’ai découvert le rôle de `@onfocusout` et de `OnFocusOutAsync`.

J’ai également dû retravailler ma PR à la suite des suggestions de Copilot validées par les maintainers, tel que décrit dans la section Solution ci-dessus.

