# Yii2-GTreeTable

Yii2-GTreeTable jest rozszerzeniem frameworka Yii 2, które z jednej strony stanowi opakowanie pluginu [GTreeTable](https://github.com/gilek/GTreeTable), z drugiej zapewnia jego obsługę od strony serwerowej.

Dzięki oprogramowaniu możliwa staje się realizacja operacji typu CRUD oraz zmiana położenia węzła wewnątrz drzewa.

Działanie aplikacji można przetestować na [demo projektu](http://gtreetable.gilek.net).

![](http://gilek.net/images/gtt2-demo.png)

## Instalacja

Instalacja odbywa się za pomocą menadżera [Composer](https://getcomposer.org).

W konsoli wpisz polecenie:

```
php composer.phar require  "gilek/Yii2-GTreeTable *"
```

lub dodaj następującą linijkę do sekcji `require` pliku `composer.json` Twojego projektu:

```
"gilek/Yii2-GTreeTable": "*"
```

## Minimalna konfiguracja

1. Tworzymy tabelę do przechowywania węzłów:

    ``` sql
    CREATE TABLE `tree` (
      `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
      `root` INT(10) UNSIGNED DEFAULT NULL,
      `lft` INT(10) UNSIGNED NOT NULL,
      `rgt` INT(10) UNSIGNED NOT NULL,
      `level` SMALLINT(5) UNSIGNED NOT NULL,
      `type` VARCHAR(64) NOT NULL,
      `name` VARCHAR(128) NOT NULL,
      PRIMARY KEY (`id`),
      KEY `root` (`root`),
      KEY `lft` (`lft`),
      KEY `rgt` (`rgt`),
      KEY `level` (`level`)
    );
    ```

2. Dodajemy węzeł główny:

    ``` sql
    INSERT INTO `tree` (`id`, `root`, `lft`, `rgt`, `level`, `type`, `name`) VALUES (1, 1, 0, 1, 0, 'default', 'Węzeł główny');
    ```

3. Tworzymy nową klasę [aktywnego rekordu](http://www.yiiframework.com/doc-2.0/guide-db-active-record.html) na postawie tabeli z punktu 1. Istotne jest aby dziedziczyła z klasy `gilek\gtreetable\models\TreeModel`:

    ``` php
    class Tree extends \gilek\gtreetable\models\TreeModel {
    
      public static function tableName()
      {
        return 'tree';
      }
    }
    ```

4. Tworzymy nowy kontroler lub dodajemy do istniejącego akcje:

    ``` php
    use app\models\Tree;
    
    class TreeController extends \yii\web\Controller
    {        
      public function actions() {
        return [
          'nodeChildren' => [
            'class' => 'gilek\gtreetable\actions\NodeChildrenAction',
            'treeModelName' => Tree::className()
          ],
          'nodeCreate' => [
            'class' => 'gilek\gtreetable\actions\NodeCreateAction',
            'treeModelName' => Tree::className()
          ],
          'nodeUpdate' => [
            'class' => 'gilek\gtreetable\actions\NodeUpdateAction',
            'treeModelName' => Tree::className()
          ],
          'nodeDelete' => [
            'class' => 'gilek\gtreetable\actions\NodeDeleteAction',
            'treeModelName' => Tree::className()
          ],
          'nodeMove' => [
            'class' => 'gilek\gtreetable\actions\NodeMoveAction',
            'treeModelName' => Tree::className()
          ],            
        ];
      }

      public function actionIndex() {
        return $this->render('gilek\gtreetable\views\widget');
      }
    }
    ```

5. W pliku konfiguracyjnym dodajemy odwołanie do folderu z tłumaczeniami:

    ``` php
    'i18n' => [
      'translations' => [
        'gtreetable' => [
          'class' => 'yii\i18n\PhpMessageSource',
          'basePath' => 'gilek\gtreetable\messages',                     
        ]
      ]
    ]
    ```  

## Konfiguracja

### Akcje

Wszystkie akcje z lokalizacji `gilek\gtreetable\actions` posiadają parametry:

 + `$access` (string) - nazwa jednostki autoryzacyjnej do weryfikacji. 

    Przed wykonaniem akcji możliwe jest sprawdzenie, czy użytkownik posiada dostęp do aktualnej podstrony. Więcej informacji na ten temat znajdziesz w [przewodniku Yii 2.0](http://www.yiiframework.com/doc-2.0/guide-security-authorization.html#role-based-access-control-rbac),

  + `$treeModelName` (TreeModel) - odwołanie do modelu danych dziedziczącego z `gilek\gtreetable\models\TreeModel` (patrz [Minimalna konfiguracja](#minimalna-konfiguracja) punkt 1).
 
Dodatkowo w przypadku akcji usuwania węzła `gilek\gtreetable\actions\NodeDeleteAction` możliwe jest zdefiniowanie parametru:

  + `$dependencies` (array) - w sytuacji, gdy model powiązany jest z innymi danymi, możliwe jest wykonanie pewnych dodatkowych operacji. 
    
    Parametr powinien być tablicą, której klucze są nazwami relacji modelu zdefiniowanego w parametrze `$treeModelName`, z kolei wartości winny być anonimowymi funkcjami zwrotnymi.

    Całość najlepiej obrazuje poniższy przykład, który wygeneruje błąd, w momencie, gdy usuwany węzeł będą miał jakieś powiązana w relationsA:

    ``` php
    [
        'relationsA' => function($relationsA, $model) {
            if (count($relationsA) > 0) {
                throw new HttpException('500');
            }
        }
    ]
    ```

### Model 

Obsługa struktury drzewiastej w bazie danych oparta jest na modelu [Nested set model](http://en.wikipedia.org/wiki/Nested_set_model). 

Abstrakcyjna klasa `gilek\gtreetable\models\TreeModel` zapewnia obsługę w/w modelu po stronie PHP, definiuje reguły walidacyjne oraz dostarcza dodatkowe metody. Jej konfiguracji można dokonać poprzez właściwości:
    
  + `$hasManyRoots` (boolean) - określa, czy możliwe jest tworzenie więcej niż jednego węzła głównego. Domyślnie `true`,

  + `$leftAttribute` (string) - nazwy kolumny przechowującej lewą wartość.  Domyślnie `lft`,

  + `$levelAttribute` (string) - nazwy kolumny przechowującej poziom węzła. Domyślnie `level`,

  + `$nameAttribute` (string) - nazwy kolumny przechowującej etykietę węzła. Domyślnie `name`,

  + `$rightAttribute` (string) - nazwy kolumny przechowującej prawą wartość. Domyślnie `rgt`,

  + `$rootAttribute` (string) - nazwy kolumny przechowującej odwołanie od ID węzła głównego. Domyślnie `root`,

  + `$typeAttribute` (string) - nazwy kolumny przechowującej typ węzła . Domyślnie `type`.

### Widok 

Klasa widoku `gilek\gtreetable\views\widget` zawiera gotową konfigurację [operacji CUD](https://github.com/gilek/GTreeTable/blob/2.0a/README_pl.md#cud) wraz z odwołaniem do [źródła węzłów](https://github.com/gilek/GTreeTable/blob/2.0a/README_pl.md#param-source). Nie ma konieczności, aby z niej korzystać, jednak może okazać się bardzo pomocna, w przypadku prostych projektów. 
Całość można dostosować do swoich potrzeb poprzez parametry:

  + `$controller` (string) - nazwa kontrolera, w którym zdefiniowano akcje (patrz [Minimalna konfiguracja](#minimalna-konfiguracja) punkt 4). Domyślnie przyjmowana jest wartość z której nastąpiło wywołanie widoku `gilek\gtreetable\views\widget`,

  + `$options` (array) - opcje przekazywane bezpośrednio do pluginu GTreeTable,

  + `$routes` (array) - w przypadku, gdy poszczególne akcje ulokowane są w różnych kontrolerach lub ich nazwy są odmienne w stosunku do przedstawionych w punkcie 4 rozdziału [minimalna konfiguracja](#minimalna-konfiguracja), wówczas konieczne staje się ich zdefiniowanie. 

    Wymaganą strukturę danych, najlepiej obrazuje poniższy przykład:

    ``` php
    [
      'nodeChildren' => 'controllerA/source',
      'nodeCreate' => 'controllerB/create',
      'nodeUpdate' => 'controllerC/update',
      'nodeDelete' => 'controllerD/delete',
      'nodeMove' => 'controllerE/move'
    ]
    ```

  + `$title` (string) - definiuje tytuł strony, gdy widok jest wywoływany bezpośrednio z poziomu akcji (patrz [Minimalna konfiguracja](#minimalna-konfiguracja) punkt 4).

### Widżet 

Głównym zadaniem widżetu `gilek\gtreetable\GTreeTableWidget` jest wygenerowanie parametrów konfiguracyjnych pluginu GTreeTable oraz dołączenie wymaganych plików. W przypadku braku kontenera, odpowiada on również za jego stworzenie. Klasa posiada następujące właściwości:

  + `$assetBundle` (AssetBundle) - parametr umożliwia nadpisane domyślnego pakietu AssetBundle tj. `GTreeTableHelperAsset`,

  + `$columnName` (string) - nazwa kolumny tabeli. Domyślna wartość `Name` pobierana jest z pliku tłumaczeń,

  + `$htmlOptions` (array) - opcje HTML kontenera, renderowane w momencie jego tworzenia (paramert `$selector` ustawiony na null),

  + `$options` (array) - opcje przekazywane bezpośrednio do pluginu GTreeTable,

  + `$selector` (string) - selektor jQuery wskazujący kontener drzewa (tag `<table>`). Ustawienie parametru na wartość null spowoduje automatyczne wygenerowanie tabeli. Domyślnie `null`,

## Ograniczenia

Yii2-GTreeTable korzysta z rozszerzenia [Nested Set behavior for Yii 2](https://github.com/creocoder/yii2-nested-set-behavior), które na obecną chwilę (wrzesień 2014) ma pewnie ograniczenia odnośnie kolejności elementów głównych (węzły, których poziom = 1). 

W przypadku dodania lub przesunięcia węzła jako element główny, wówczas zostanie on zawsze ulokowany, po ostatnim elemencie tego stopnia. W związku z czym, kolejność wyświetlanych węzłów głównych, nie zawsze ma swoje odwzorowanie w bazie danych.