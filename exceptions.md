# Использование исключений

Продолжительность разработки и поддержки приложений обуславливает постоянное изменение модулей и их интерфейсов.
В свою очередь изменение интерфеса модуля может оказывать влияние на клиенткий код, который его использует.
Важную роль в этом играет подход к спецификации и обработке исключительных ситуаций.

## Введение

Интерфейс модуля определяется множеством предоставляемых им общедоступных методов.
Каждый из этих методов имеет сигнатуру, которая включает его имя, список параметров и т.д.
Кроме того, сигнатура метода может включать список исключений, которые могут возникнуть в его работе.

Язык программирования Java предоставляет развитые средства для работы с исключениями.
В частности он дает возможность определять те исключения, которые должны быть обработаны при вызове метода в обязательном порядке.
Однако, необходимость и полезность применения такого подхода является спорной.

## Иерархии исключений

В языке Java исключения, обработка которых является обязательной, называются проверяемыми (checked).
Такие исключения представляются в виде класса, который расширяет класс `Exception` или один из его подклассов (за исключением `RuntimeException`).

```java
public class NotFoundException extends Exception {
}
```

Если метод может выбросить проверяемое исключение, то оно должно быть описано в его сигнатуре.

Рассмотрим простой пример интерфейса для хранилища документов.
В этом примере метод `find` может выбросить исключение `NotFoundException`, если документ не был найден.
Также методы `find` и `save` могут выбросить исключение `NotAllowedException`, если пользователь не имеет прав для выполнения операции.

```java
public interface DocumentsRepository {
    Document find(String name) throws NotFoundException, NotAllowedException;
    Document save(Document document) throws NotAllowedException;
}
```

Пусть следующий метод используется в другом модуле для открытия документа.
Вначале в нем выполняется попытка найти документ с заданным именем в хранилище.
Если документ не был найден, то выполняется попытка создать пустой документ.

```java
public Document open(String name) {
    try {
        try {
            return documentsRepository.find(name);
        } catch (NotFoundException exception) {
            Document empty = DocumentFactory.createEmpty(name);
            return documents.save(empty);
        }
    } catch (NotAllowedException exception) {
        return null;
    }
}
```

Недостатком данной реализации является способ обработки исключения `NotAllowedException`.
Более правильной была бы передача этого исключения или его обертывание.
Тем не менее, этот пример иллюстрирует то, что проверяемое исключение может не быть значимым в определенным контексте.
Однако, при этом, требовать определенных усилий для обработки.

Допустим, что в дальнейшем понадобилась возможность работы с хранилищами, которые доступны только для чтения.
Для таких хранилищ вызов метода `save` является исключительной ситуацией, в случае которой выбрасывается проверяемое исключение `NotSupportedException`.

```java
public interface DocumentsRepository {
    Document find(String name) throws NotFoundException, NotAllowedException;
    Document save(Document document) throws NotSupportedException, NotAllowedException;
}
```

Добавление нового проверяемого исключения в сигнатуру метода `save` ведет к несовместимости с ранее разработанным кодом открытия документа.
Причиной этого является отсутствие обязательного обработчика для исключения `NotSupportedException`.

Чтобы избежать описанной проблемы, необходимо использовать иерархии исключений.
В качестве базового можно добавить исключение `RepositoryException`.

```java
public abstract class RepositoryException extends Exception {
}
```

Базовое исключение может быть абстрактным, чтобы указать на необходимость создания конкретных исключений для различных исключительных ситуаций.
Все остальные исключения должны прямо или косвенно наследовать базовое исключение.
В результате интерфейс доступа к хранилищу будет иметь следующий вид.

```java
public interface DocumentsRepository {
    Document find(String name) throws RepositoryException;
    Document save(Document document) throws RepositoryException;
}
```

Соответствующим образом изменится и клиентский код.

```java
public Document open(String name) {
    try {
        try {
            return documentsRepository.find(name);
        } catch (NotFoundException exception) {
            Document empty = DocumentFactory.createEmpty(name);
            return documents.save(empty);
        }
    } catch (RepositoryException exception) {
        return null;
    }
}
```

Код открытия документа различает ситуацию, когда документ отсутствует в хранилище.
В то же время, остальные исключительные ситуации не существенны в данном контексте.
Теперь уточнение методов работы с хранилищем не будет оказывать существенного влияния на этот код.

То есть, объединение исключений в одну иерархию позволяет упростить написание и поддержку клиентского кода.
Если исключения являются частью одной иерархии, то явное перечисление нескольких исключений в сигнатуре метода становится избыточным.
Достаточно указать базовое исключение в сигнатуре метода и конкретные исключения в документации к нему.

## Непроверяемые исключения

По мере развития модуля его интерфейс может расширяться и дополняться.
Например, может потребоваться добавить в интерфейс хранилища новый общедоступный метод.
При этом, если базовое исключение в иерархии является проверяемым, то оно должно быть добавлено в сигнатуру этого метода.
Оно должно быть добавлено даже в том случае, если добавленный метод на данный момент не выбрасывает никаких исключений.
Это позволит избежать проблем с клиентским кодом в будущем, когда выбрасываение исключений может стать необходимым.

В результате спецификация исключений в сигнатуре методов теряет информативность.

```java
public interface DocumentsRepository {
    Document index() throws RepositoryException;
    Document find(String name) throws RepositoryException;
    Document save(Document document) throws RepositoryException;
}
```

Для того, чтобы избежать этого, базовой исключение должно быть объявлено как непроверяемое (unchecked).
В Java непроверяемое исключение наследуется от класса `RuntimeException` или его подкласса.

```java
public abstract class RepositoryException extends RuntimeException {
}
```

В результате упрощается сигнатура методов интерфейса, а описание выбрасываемых исключений становится частью документации.
Кроме того, упрощается и клиентский код, который может обрабатывать исключения в подходящем конктесте без дополнительных затрат.

```java
public Document open(String name) {
    try {
        return documentsRepository.find(name);
    } catch (NotFoundException exception) {
        Document empty = DocumentFactory.createEmpty(name);
        return documents.save(empty);
    }
}
```

## Выводы

Использование проверяемых исключений в Java не дает значимых преимуществ перед непроверяемыми.
Кроме того, их использование может усложнить написание и поддержку клиентского кода.
Наилучшим подходом на данный момент для Java является использование иерархий исключений, в которых базовое исключение является непроверяемым.