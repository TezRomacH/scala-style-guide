
# KINOPLAN scala style guide
## Оглавление
* Наименования
* Вопросы синтаксиса
* Вызовы функций
* Общие принципы разработки в стиле Model-View-Controller на Play Framewok
* Рефакторинг кода

## Наименования
В большинстве случаев используется `camelCase`, при этом слова пишутся полностью.
Это верно для переменных `var`, значений `val` (если они не являются константами), функций `def`, параметров функций

Примеры:
```scala
val usersWithCinemas = ...

def createNote(releaseId: Int) = ...

var counter = ...
``` 

**Константы** именуются в `UPPER_SNAKE_CASE`.
Примеры:
```scala
val MAXIMUM_PASSWORD_LENGTH = 32
val SOCKET_ADD_EVENT = "add"
val CINEMA_TYPES = List("cinema", "manager")
```

Классы, case-классы, абстрактные классы, трейты, объекты именуются в `PascalCase`.

Примеры:
```scala
class RepertoireDAO (  
    ...
)

case class Note(
    ...
)

trait NoteJson {  
    ...
}

object Note extends NoteJson {
    ...
}
```
Типы, при описании значений, функций и констант, рекомендуется скрывать, для визуального уменьшения кода, если из него понятно, какой тип будет возвращен.
Очивидными являются возвращаемые типы при использовании методов `.toList`, `.toOption`, `.toMap`, при использовании значений
`val SOCKET_ADD_EVENT = "add"`,

`val MAXIMUM_PASSWORD_LENGTH = 32`,

`val FULL_CINEMA_TYPES = List(CINEMA, MECHANIC, MANAGER, ADMINISTRATOR)`.


В ответе сервера в JSON и в базах данных используется `snake_case`.

```json
{
	"cinema_id": 1803,
	"is_on_sale": true
}
```

Имена модулей одним словом маленькими буквами
`package seance`.

Для значений типа `Option[_]` в конце имени добавляется `O`
```scala
def list(releaseIdO: Option[Int]) = ...
```

## Вопросы синтаксиса
Переменных `var` избегаем, использование только в *крайне* редких случаях, если мы используем библиотеки, написанные для `Java` и если без мутабельности не обойтись.

Если предстоит работа с несколькими опциональными значениями или списками, то стараемся не выделять их в отдельные переменные, а использовать цепочечные конструкции:
```scala
cinemaRightsService.find(cinemaId).map { cinemaRights =>
	noteService.find(cinemaId, releaseId).map { note =>
		...
	}
}
```
При использовании цепочечных функций `map`, `flatMap`, `filter` и т.д. руководствуемся следующими правилами:
*  Если внутри используется лишь вызов одной друго функции, то не выделяем переменную и используем скобки: `.foreach(println)`.
* Если внутри функции используется лишь одно поле, то используется вызов, через `_` и круглые скобки: `.filter(_.id == cinemaId)` .
* Если используется значение, но функция умещается в одну короткую строчку, выделяем переменную и используем круглые скобки: `.map(proposals => Json.toJson(repertoires)(Repertoire.listWrites(proposalsWeeks = proposals)))`.
* В остальных случаях, когда функция получается больше, чем на одну строку, выделяем переменную и используем фигурные скобки, при этом оставляя переменную на той же строке:  
 ```scala
cinemaRightsService.find(cinemaId).map { cinemaRights =>
	noteService.find(cinemaId, releaseId).map { note =>
		...
	}
}
```
* Для partiotialFunctions всегда фигурные скобки! Если есть всего одна конструкция `case ... =>`, то, как и в предыдущем случае, оставляем ее на той же строке, если же есть несколько конструкций `case`, то все с новых строк:
```scala
repertoires.groupBy(_.releaseId).flatMap { case (releaseId, repertoireList) =>
	...
}
```
```scala
appsToUpdate match {
    case Nil => NotModified
    case apps: List[(PosterApp, Apk)] => Ok(Json.toJson(apps)
}
```

Отдельно для пары `flatMap` и `filter`, если в цепочке связаны ≥ 3 `flatMap`, то возможно следует вынести это в `for`.

Для соединения цепочек, используется вариант вызова функцию через точку:
```scala
someService.flatMap { someVal =>
	innerService.map(_ + someVal)
}.getOrElse(SOME_CONSTANT)
```
Исключением являются функции над объектом, принемающие объект того же типа: `orElse`, `andThen` `keepAnd`, `and (в Reads)` etc. Для них используется инфиксная нотация.

```scala
(someO orElse otherO orElse).getOrElse( ... )
```

При использовании функций в инфиксной нотации, если их ≥ 3, то то все записываются на новой строчке, сама функция остается в конце строки, а все применяемые значения выравниваются по ширине, как равные!
Примеры:
```scala
// Плохо
def update(noteId: ObjectId) = (
    authUtils.authenticateAction()
    andThen authUtils.noteAction(noteId)
    andThen canModifyNote
) { request => ...

// Хорошо
def update(noteId: ObjectId) = (
    authUtils.authenticateAction() andThen
    authUtils.noteAction(noteId) andThen
    canModifyNote
) { request => ...
```
```scala
// Плохо
(
	Reads.pure(new ObjectId) and
	    Reads.pure(userId) and
	    (__ \ "message_id").readNullable[String] and
	    Reads.pure(false)
) (SomeModel.apply _)

// Хорошо
(
	Reads.pure(new ObjectId) and
	Reads.pure(userId) and
	(__ \ "message_id").readNullable[String] and
	Reads.pure(false)
) (SomeModel.apply _)
```

При использовании конструкции `if (...) { ... } else { ... }` фигурные скобки ставятся всегда, если только это не присваивание и все умещается на одну строку:
```scala
val noteEvent = if (note.deleted) Note.SOCKET_ADD_EVENT else Note.SOCKET_UPDATE_EVENT
```
Во всех остальных случаях
```scala
val some = if (file.length <= 1.mb) {
	...
} else {
 ...
}
```

```scala
if(user.isOurEmployee) {
	...
} else {
	...
}
```

Все импорты делаем в начале файла.

### Пробелы
После `if`, перед скобками ставиться пробел: `if (someBoolean) { ... }`
После указания типа ставиться пробел, но не перед двоеточием:
```scala
// Плохо
val cinemaIds : List[Int]

// Плохо
val cinemaIds :List[Int]

// Плохо
val cinemaIds:List[Int]

// Хорошо
val cinemaIds: List[Int]
```

##  Вызовы функций
Метод `apply` всегда вызывается в круглых скобках, поэтому например создание инстанса case-класса: `Note(cinemaId, releaseId, text)`.
Если нужно передать в case-класс ≥ 4 параметра, то каждый параметр записывается на новой строке.

Функции без параметров пишутся и вызываются без скобок:
```scala
// определение функции без параметров
def list = authUtils.authenticateAction() { ...}

// использование функции без параметров
someCursorInDAO.toList
```

Если при вызове функции нужно передать какое-то магическое значение, то внутри мы поясняем параметр через `=`:
```scala
.getSettings(cinemaId, releaseId, withUpdate = true)
```

Для функций не возращающих классы, используется синтаксис `: Unit = ...`:
`def updateCoverUrl(id: Int, url: String): Unit = {`

Процедурный синтаксис `def updateCoverUrl(id: Int, url: String) {` был признан в Scala, как устаревший, [и от него собираются избавится.](https://github.com/lampepfl/dotty/blob/master/docs/docs/reference/dropped/procedure-syntax.md)

## Общие принципы разработки в стиле Model-View-Controller на Play Framewok
### Model
Модель только отображает набор переменных в класс, знает о существовании других моделей, может их использовать в своем конструкторе. Модель также может содержать набор методов, использующие её параметры. Примеры: `val id = _id.toString`, `def hasText = text.nonEmpty`, `def isDeleted = deleted == 1`, `val hallIds: List[Int] = halls.map(_.id)`.

В случае, если нужно преобразовать сами параметры модели, то мы не используем `var`, а создаем новую модель, на основе предыдущей, с помощью функции `copy`. Причем, такая функция должна начинаться со слова `with` и пояснять, что именно изменено:
`def withNewStatus(newStatus: String): SomeModel = this.copy(status = newStatus)`.

Модели оформляются, как `case class`. В моделях не должно быть использования сервисов, DAO. На каждую модель отдельный файл. В одном файле модели может находиться case-класс, object и trait для `Writes/Reads`.

Trait для `Writes/Reads` оформляется, как имя класса + `Json`:
```scala
case class SomeModel(...)

trait SomeModelJson { ... }

object SomeModel extends SomeModelJson { ... }
```

Все константы необходимые классу, пишутся только в его object. 

`Writes` создаем в нотации функции: `val writes: Writes[Note] = (note: Note) => { ... }`.

### Data Access Object (DAO)
DAO ничего не знает о других DAO, ничего не знает о сервисах и контроллерах. Единственное ее назначение, отображать объекты из какой-либо таблицы БД в scala-класс. Если для одного и того же класса нужно получать даныне из разных таблиц или даже из разных БД (MySQL и MongoDB, тогда создаются два отдельных DAO.

DAO создаются как синглтоны (`@Singleton`) и инжектятся в Сервисы.

### Service
Сервисы, в отличие от DAO, уже знают о существовании других сервисов и могут их использовать, с помощью аннотации `@Inject`.

### Controllers
Контроллеры не знают о DAO, видят только модели и сервисы.
К каждому роуту в контроллере обязательно должны быть свежие документации:
```
@api
@apiName
@apiGroup
@apiParam
@apiParamExample
@apiSuccessExample
```

Все ошибки, должны лежать в `utils.Error` и иметь свой номер.
Внутри `Result` не должно быть дополнительных преобразований, фильтрации и прочего.
```scala
// плохо
Ok(
	cinemas.map { cinema =>
		cinema.filter(_.isFondKino)....
			....map(Json.toJson)
	}
)

// хорошо
Ok(Json.toJson(cinemas)(Cinema.writes))
```
В случае, если все выполнилось правильно, но отправить нечего, то отправляем `NoContent`, не `Ok` и уж тем более не `Ok(Json.toJson("status" -> "ok"))`!

Имена методов в контроллере стараемся делать унифицированными для всего проекта. Вот список потенциальных имен для функций в контроллере:
```scala
def list = // получить все объекты в контроллере
def getSomething = // получить какой-то объект (объекты). Например getMessages, getPoster
def getBySomeId = // получить объект (объекты) по какому-то ключу
def create = 
def update(id: ...) = 
def delete(id: ...) = 
```

Когда в контроллере, есть необходимость получить объект по `id`  и проверить права доступа к нему, то в `actions` создается отдельный request с этим объектом, отдельный `ActionRefiner` и, если нужно уточнить права доступа, то и отдельный `ActionFilter`.
```scala
class RequestWithNote[A](val note: Note, val user: User, request: AuthenticatedRequest[A])
  extends RequestApi[A](request)
```
```scala
def noteAction(noteId: ObjectId): ActionRefiner[AuthenticatedRequest, RequestWithNote]
```

## Рефакторинг кода
Правило бойскаута: 
> Оставь место стоянки чище, чем оно было до твоего прихода.

Из книги Роберта Мартина **Чистый код**:
> Если мы все будем оставлять свой код чище, чем он был до нашего прихода, то код попросту не будет загнивать. Чистка не обязана быть глобальной. Присвойте более понятное имя переменной, разбейте слишком большую функцию, устраните одно незначительное повторение, упростите сложную цепочку условий.
Представляете себе работу над проектом, код которого улучшается с течением времени? Но может ли профессионал позволить себе нечто иное? Разве постоянное совершенствование не является неотъемлемой частью профессионализма?

Чтобы код не накапливал легаси-код, если ты добавляешь новые методы к класс, или новый роут в контроллере, поправь код-стайл **во всем файле!** Это не так страшно, просто действуй в соответствии с этим файлом: расставь правильно скобочки, посмотри на названия переменных, нет ли каких-нибудь преобразований в `Result`. 
При этом никто не требует от тебя переписать логику всего кода! Если возникла потребность обновить логику старого кода, то в Git осздается отдельная ветка `refact/<что-то, что требует рефакторинга>`, более подробно будет в файле `git.md` в этом репозитории.
