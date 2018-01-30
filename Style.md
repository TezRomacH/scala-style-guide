

# KINOPLAN scala style guide
## Оглавление
* Наименования
* Вопросы синтаксиса
* Вызовы функций
* Комментарии
* Вопросы использования Play Framework

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
* Для partiotialFunctions всегда фигурные скобки!
* В остальных случаях, когда функция получается больше, чем на одну строку, выделяем переменную и используем фигурные скобки, при этом оставляя переменную на той же строке:  
 ```scala
cinemaRightsService.find(cinemaId).map { cinemaRights =>
	noteService.find(cinemaId, releaseId).map { note =>
		...
	}
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

Если при вызове функции нужно вызвать какое-то магическое значение, то внутри мы поясняем параметр через `=`:
```scala
.getSettings(cinemaId, releaseId, withUpdate = true)
```

## Комментарии
Мы не пишем комментариев в коде, считается, если куску кода нужно дополнительное пояснение, то этот кусок недостаточно хорошо написан и стоит, наверняка, переписать его.
Исключения составляют лишь документирующие комментарии к контроллерам, к ним нужны обязательно доки:
```
@api
@apiName
@apiGroup
@apiParam
@apiParamExample
@apiSuccessExample
```

## Вопросы использования Play Framework
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

В моделях не должно быть использования сервисов. На каждую модель отдельный файл. В одном файле модели может находиться case-класс, object и trait для `Writes/Reads`.

`Writes` создаем в нотации функции:
`val writes: Writes[Note] = (note: Note) => { ... }`. 
