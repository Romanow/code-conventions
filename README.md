[![License](https://img.shields.io/github/license/Romanow/code-conventions)](https://github.com/Romanow/code-conventions/blob/master/LICENSE)

# Code Conventions

## Code Style

За основу разметки в коде
берем [Google Java Code Style](https://google.github.io/styleguide/javaguide.html).

Описание [Code Style](.idea/codeStyles/Project.xml) кладем в папку `.idea/codeStyle/Project.xml` и
там же рядом создаем файл [`codeStyleConfig.xml`](.idea/codeStyles/codeStyleConfig.xml), чтобы Code
Style использовался из проекта:

```xml

<component name="ProjectCodeStyleConfiguration">
    <state>
        <option name="USE_PER_PROJECT_SETTINGS" value="true"/>
    </state>
</component>
 ```

Папку `.idea` убираем из `.gitignore` добавляем в индекс. В папке `.idea` есть свой
[`.gitignore`](.idea/.gitignore), где прописано, что user-specific файлы не попадали в git.

Если папка `.idea/` в `.gitignore`, то нужно `.idea/codeStyles/*` и `.idea/.gitignore` добавить
вручную:

```shell
$ git add -f .idea/.gitignore
$ git add -f .idea/codeStyles/codeStyleConfig.xml .idea/codeStyles/Project.xml
```

## Соглашение по коду

### Общие требования

1. Если Intellij Idea подсвечивает код, то его нужно исправить.
2. Переопределенные методы всегда помечаем аннотацией `@Override`.
3. Можно использовать `var` для описания локальных переменных, если тип переменной очевиден из
   правого выражения.
   ```java
   public static void main(String[] args) {
       var users = List.of("Alex", "Andrew", "Max", "Anton", "Gleb");
       for (var user: users) {
            System.out.println(user);
       }
   }
   ```
4. Если `var` не используется, то используем `diamond operator <>`
   ```java
   private Map<String, Pair<Integer, String>> ratingByUsers = new HashMap<>();
   ```
5. Для создания список используем `List.of()`, для создания map `Map.of()`. При этом создаются
   **readonly** коллекции.
   ```java
   private List<String> users = List.of("Alex", "Andrew", "Max", "Anton", "Gleb");
   private Map<String, Integer> ratingByUsers = Map.of("Alex", 1, "Max", 2, "Andrew", 3);
   ```
6. Не используем checked-исключения, только наследников от `RuntimeException`.
7. Крайне нежелательно перехватывать обобщенные исключения `Throwable`.
8. Никогда нельзя перехватывать исключения типа `Error`.
9. Использовать пробелы вместо табуляции.
10. Не использовать в заголовке класса `@author` (отключить в Intellij Idea). Они указывают кто
    создал файл, но после этого много народу могут менять этот файл. Для понимания, кто автор
    изменений в файле, использовать `git annotate`.
11. Метод не должен занимать больше 3 экранов. Если метод больше, то следует разносить его на более
    мелкие методы.
12. Все magic numbers, constant strings выносим в именованные константы. Константа
    описывается `(public|private) static final`.
13. Не использовать XML Dom, XPath для навигации по объектам.
14. Нельзя использовать хранимые процедуры.
15. Используем YAML для всех конфигураций, т.к. на больших файлах это нагляднее.
16. Стараемся использовать `@ConfigurationProperties`, т.к. это позволяет проверять заполненность
    данных на этапе запуска приложения.
17. Строить бизнес-логику на exception можно, например, если в процессе выполнения были получены
    какие-то данные, работа с которыми невозможна.
18. Если метод должен выполнить проверку, то лучше, чтобы он возвращал `true` / `false`, а
    вышестоящий код мог сам решить что ему делать с этим.
    ```java
    void unzip(@NotNull String str) {
        if (!isAcceptableForUnzip(str)) {
            throw new RuntimeException("String '" + str + "' not acceptable for unzip (size not multiple by 4)");
        }
    }
    boolean isAcceptableForUnzip(@Nonnull String str) {
        return str.length() % 4 == 0;
    }
    ```
19. Если есть параметризованные сложные `SQL`, то для их построения использовать `Criteria API`, не
    клеить их руками через конкатенацию.
20. Контролеры максимально простые, вся бизнес логика внутри сервисов. Объекты `HttpServletRequest`,
    `HttpServletResponse`, `SecurityContextHolder` и т.п. могут использоваться только в слое `web`,
    а в сервисы пробрасываются уже результат, который мы из них получили. В противном случае мы
    мешаем логику представления с бизнес-логикой.
21. Если проверка валидации требует доступ к базе данных, то ее нужно переносить в слой сервисов,
    иначе реализовывать через аннотации JSR303 или кодом в контроллере.
22. Аннотацию `@SneakyThrows` можно использовать в _тестах_ в любом месте.
23. В _коде_ можно использовать аннотацию `@SneakyThrows` только для _приватных методов_.
    `@SneakyThrows` нельзя использовать в методах, которые оборачиваются другими аннотациями
    Spring (например, `@Transactional`), потому что она маскирует checked exception, делая возможным
    не описывать его в сигнатуре метода `throws`. Spring создает Dynamic Proxy ожидая корректную
    сигнатуру метода. [Объяснение](https://stackoverflow.com/a/56508336)<br/>
    Если требуется использовать `@SnekyThrows`, то выносим код в private метод:
    ```java
    @SneakyThrows
    private void deleteFile(@NotNull String path) {
        Files.deleteIfExists(path);
    }
    ```
24. Не использовать `printStackTrace()`, т.к. он не попадает в лог (ELK).
    ```java
    logger.error("", e);
    ```

### Общая структура проекта

В корне проекта лежит главный файл`@SpringBootApplication`. Пакеты разбиваются по слоям:

* `web` – `@Controllers`, `@Converter`, `@RestControllerAdvice` и т.п.
* `services` – бизнес логика и сервисы.
* `entities` – доменные сущности, помеченные аннотацией `@Entity`.
* `repositories` – `JpaRepository` и DAO классы (c `@PersistenceContext`).
* `config` – `@Configuration` классы, каждая конфигурация по слоям выносится в отдельный класс. Т.е.
  bean и настройки, относящиеся к web, должно находится в WebConfiguration, но не обязательно только
  в нем одном. Можно разносить на более специфичные конфигурации. Главное не мешать конфигурации
  воедино, например конфигурация `@EnableJpaAuditing` не должна лежать в той же самой конфигурации,
  что и создание `WebClient`.
* `models` – классы моделей.
* `mappings` – mapstruct mappers, так же здесь могут находиться утильные сервисы для маппинга.
* `exceptions` – исключения.
* `utils` – разные утильные классы. Обычно это статические helper или `@Component` утилитарного
  характера.

Если в сервисе выделяется больше одной доменной области, например, `User` и `Wallet`, то все
классы (`web`, `mappings`,`models`, `entities`, `repositories`, `services`), связанные с ними,
должны находится в отдельном пакете user и wallet соответственно.

Для того чтобы сервис (`@Service`) был изолированный, он должен взаимодействовать только с DAO и
репозиториями из своего домена. Т.е. если нам в `WalletService` нужно получить пользователя, то мы
должны использовать`UserService`, а не работать напрямую с `UserRepository`. Иначе нарушается
_Single Responsibility_ принцип и сильно усложняются unit тесты.

Если есть какие-то общие классы, сервисы, то они выносятся в пакет `common`.

### Правила описания моделей

* `@Entity` не должен использоваться напрямую в контроллере.
* Для разных контроллеров описываем свои модели и разделяем их для запросов и ответов.
* В моделях, которые используются непосредственно в контроллерах, можно использовать суффиксы
  `Request` и `Response`.
* Модели (не `@Entity`), являющиеся проекциями доменных сущностей, именуются как сущности, но с
  суффиксом `Request`, `Response`, `Data`, `Info`, `Item`. Суффикс `DTO` не используем.
* Фильтрацию отображаемых контроллером полей модели можно провести и с использованием `JsonView`,
  тогда не нужно будет создавать синтетические модели с их mappings.
* В именовании моделей можно использовать суффикс `List`, только если модель является наследником от
  коллекции. Но лучше использовать просто множественное число в именовании.
* В модели допускается описывать несколько конструкторов, но для корректной сериализации обычно
  должен быть дефолтный (пустой) конструктор.
* В описании моделей всегда используем `Boolean`, т.к. если поле описано как `boolean` и в запросе
  приходит `null`, мы получаем `NullPointerException` с очень странным stacktrace.
* `getter` и `setter` не должны содержать никакой логики (в том числе ленивая инициализация), т.к.
  модель является лишь представлением данных при передаче.
* В моделях допускается inline инициализация (или в конструкторе) сложных структур данных
  `private List<String> list = new ArrayList<>()`.
* В моделях можно использовать `@Accessors(chain = true)` для сборки через chaining.
* В моделях можно использовать аннотацию `@Data` для создания `equals`, `hashCode`, `toString`. Если
  в эти методы требуется внести изменения (например убрать поле из `toString`), то переопределяем их
  руками. Использование аннотаций `@EqualsAndHashCode.Exclude`, `@ToString.Exclude` ухудшает
  читабельность.
* Если объект имеет уникальное поле (например `uid`), то `equals` и `hashCode` нужно делать от него.
  Не нужно делать `equals` и `hashCode` от `@Id` (если это поле генерируется Hibernate на этапе
  сохранения).
* Если `equals`, `hashCode`, `toString` реализуется для `@Entity`, то в них не должно быть связных
  сущностей (даже если они `fetch = EAGER`).

### Комментарии к коду

* Комментарии пишутся с большой буквы.
* Javadoc комментарии на каждый метод не нужны, они не несут полезной нагрузки и в случае изменения
  поведения метода, можно забыть их исправить, что приведет негативному эффекту: описание метода не
  соответствует реализации.
* Очевидные комментарии по коду тоже не нужны, в них нет никакого смысла:
  ```java
  public void cancelActiveProcesses(UUID groupUid) {
      // Ищем process по processesUid
      final List<Process> processes = processRepository.findByUid(groupUid);
      // Цикл по списку processes
      for (var p : processes) {
          if (p.status == ACTIVE) {}
      }
  }
  ```
* Название метода должно отражать что делает этот метод, при этом название должно оставаться
  лаконичным: `createProcess`, `updateProcessStatus`, `generateExcel` и т.п. Если в названии
  нельзя однозначно описать, что происходит в методе, то скорее всего нарушается принцип Single
  Responsibility. Не нужно пытаться запихнуть в название всю информацию о происходящих в нем
  действиях.
  ```java
  @NotNull
  @Override
  User createUser(@NotNull CreateUserRequest request) {}

  @Override
  void lockTransaction(@NotNull Transaction transaction) {}
  ```
* Если требуется закомментировать код, то нужно описать `TODO` с описанием что делает этот код и
  четкими критериями когда этот он потребуется. Желательно прикрепить задачу.
* Если в коде есть закомментированный код, которому больше 3 недель и к нему не описан `TODO`, то
  этот код можно удалять.
* Комментарии обязательно оставлять к сложным кускам кода и неочевидным решениям, например:
  ```java
  @EnableWebMvc
  @Configuration
  public class WebConfigurationLocal
          implements WebMvcConfigurer {

      @Autowired
      private ObjectMapper objectMapper;

      @Override
      public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
          // Задаем message converters для того, чтобы в MappingJackson2HttpMessageConverter использовался
          // objectMapper, который настраивается Spring application.yaml
          // StringHttpMessageConverter и ByteArrayHttpMessageConverter используются чтобы корректно отображался
          // OpenAPI endpoint, без них SwaggerWelcomeWebMvc :: openapiJson возвращает объект, обернутый в строку
          converters.add(new StringHttpMessageConverter());
          converters.add(new MappingJackson2HttpMessageConverter(objectMapper));
          converters.add(new ByteArrayHttpMessageConverter());
      }
  }
  ```

Если бы здесь не было этого комментария, причины появления этого кода быстро забылись бы и появилось
ощущение, что мы без всякой нужны переопределяем существующее поведение. Удаление этого метода
привело бы к тому, что настройка jackson сериализатора в Spring Boot
`spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS: false` (и другие) перестала бы работать и
даты стали возвращаться в виде timestamp.

### Количество параметров в методе

Если в методе больше 5 параметров, то их нужно оборачивать в объект. Большое количество параметров
может свидетельствовать о том, что:

* метод нарушает правило Single Responsibility – выполняет разные задачи.
* в случае методов в 1-2 строки и 5+ параметрами: метод слишком декомпозирован и его вызов лучше
  заменить inline-кодом.

### Использование интерфейсов

Интерфейс – это контракт класса. Он должен содержать только методы, которые описывают его _зону
ответственности_ (Responsibility). Вынесение приватных методов на уровень интерфейса нарушает
парадигму Инкапсуляции.

Интерфейс не должен содержать методы (`default` методы), крайне нежелательно описывать в нем
константы.

Для всех классов под управлением Spring, которые помечены аннотацией `@Service`и `@Repository` (т.е.
слой бизнес логики и DAO через `@PersistenceContext`), использование _интерфейсов обязательно_.

Интерфейсы не нужно описывать для утильных классов, контроллеров (`@Controller`), репозиториев
(наследников`CrudRepository`) и т.п.

Если требуется описать какую-то общую логику, то для этого используется абстрактный класс.

### Порядок описания методов

Порядок видимости:

1. `public`;
2. `protected`;
3. package private;
4. `private`.

Порядок описания методов:

1. constants;
2. static initializers;
3. final fields;
4. fields;
5. initializers;
6. constructors;
7. static methods;
8. methods;
9. inner class.

Конструкторы объявляются от большего количества параметров (более общего), к меньшему.

```java
public class GreetingPrinterImpl
        extends GreetingPrinter {

    public static final String HELLO = "Hello";
    public static final String WORLD = "world";
    private static final String GREETING;

    static {
        GREETING = HELLO + ", " + WORLD;
    }

    private final String name;
    private final Integer order;

    private Printer<String> printer;
    private CounterHolder counter;

    {
        counter = new CounterHolder();
        printer = (str, locale) -> {
            System.out.println(str);
            return str;
        };
    }

    public MyClassImpl(String name) {
        this(name, 1);
    }

    public MyClassImpl(String name, Integer order) {
        this.name = name;
        this.order = order;
    }

    @Override
    public void sayHello() {
        print(GREETING + " from " + name + " with order " + order);
    }

    protected void updatePrinter(Printer<String> printer) {
        this.printer = printer;
    }

    private void print(String greeting) {
        printer.print(greeting, Locale.getDefault());
        counter.increment();
    }

    static class CounterHolder {

        private int counter = 0;

        void increment() {
            counter++;
        }

    }

}
```

### Использование Optional

`java.util.Optional` можно использовать как возвращаемое значение, это полезно когда требуется
сделать chaining.

```java

@Override
@Transactional(readOnly = true)
public Product findByUidOrDefault(@NotNull UUID uid) {
    return dictionaryClient.findByUid(uid)
                           .map(ProductMapper::toModel)
                           .orElseGet(() -> new Product("N/A"));
}
```

Но для описания параметров метода его использовать не стоит, лучше в явном виде описать, что метод
может принимать `@Nullable`.

#### Использование префиксов в именах методов

* Если метод имеет префикс `find` или `get`, то он возвращает значение:
    * `find` – метод возвращает `Optional`, кроме случаев когда возвращается `Pageable`, `List`,
      `Map` и т.п. (они должны возвращать пустую коллекцию). Метод всегда помечается `@NotNull`.
      Если значение не найдено, то возвращается `Optional.empty()`.
    * `get` – если значение не найдено, то выбрасывается исключение.
* Аналогичное поведение в методах `getById` в `JpaRepository` и `findById` в `CrudRepository`.
* Если метод имеет префикс `build` – то обычно это некоторый утильный метод для сборки объекта из
  других объектов.
* Методы, имеющие префикс `get`, `find`, `build` никогда не могут вносить изменения в
  базу данных.
* Если это методы на уровне сервисов, то они должны иметь аннотацию
  `@Transactional(readOnly = true)` в случае, если они выполняют обращение к базе данных.

Такое именование метода запрещено (!!!):

```java

@Override
@Transactional
public MyClass findByUid(@NotNull UUID uid) {
    return myClassRepository.findByUid(uid)
                            .orElseGet(() -> {
                                var myClass = new MyClass("Hello, world", 1);
                                return myClassRepository.save(myClass);
                            });
}
```

Корректное название, для описанного выше метода `findOrCreateByUid.`

Если метод имеет префикс `create`, `update` и т.п., значит он выполняет модификацию данных.

### Использование `@Nullable`, `@NotNull`, `@Contract`

В сервисных методах описываем параметры и возвращаемые значения с помощью аннотаций `@NotNull` и
`@Nullable` (из пакета `org.jetbrains:annotations`) – они используются Intellij Idea в статическом
анализе кода (они имеют `Retention Policy = CLASS` и недоступны в runtime, в отличие от lombok
`@NotNull`, они не выбрасывают `NullPointerException`).

Для более сложного инструктирования статического анализатора можно использовать
аннотацию [`@Contract`](https://www.jetbrains.com/help/idea/contract-annotations.html).

### Constructor Injection

* Если класс под управлением Spring, то используем constructor injection. Если в классе только один
  конструктор (отличный от конструктора по-умолчанию), то его не надо помечать `@Autowired`, Spring
  поймет что используется constructor injection и будет пытаться подставить нужные beans.
* Можно использовать аннотацию `@RequiredArgsConstructor` и необходимые поля описывать как
  `private final`.
* `@RequiredArgsConstructor` не умеет работать с аннотациями на поля, т.е. если у нас в классе есть
  поля с `@Value`, `@Qualifier` и т.п., то либо описываем их как обычные поля вне конструктора, либо
  создаем конструктор руками и там описываем вне нужные аннотации.
* `@AllArgConstructor` использовать не рекомендуется по описанным выше причинам.

```java

@Service
@AllArgsConstructor
public class MyEntityServiceImpl
        implements MyEntityService {

    private final MyEntityRepository myEntityRepository;
    private final MyEntityMapper myEntityMapper;
    private final WebClient webClient;

    @Value("${request.timeout}")
    private Duration timeout;

}
```

Использование constructor injection хороший вариант, потому что при написании unit тестов сразу
понятно какие параметры нужно передавать в метод. Так же, если мы видим, что для создания класса
требуется передать большое количество параметров в конструктор, то возможно класс занимается не
только своей задачей и нарушает Single Responsibility.

### Правила именования пакетов

В именовании пакетов стараемся использовать одно слово, если это невозможно, то используем
lowerCamelCase или разделяем точками. snake_case недопустим.

```java
package ru.romanow.authorization; // OK
```

```java
package ru.romanow.empty.authorization; // OK
```

```java
package ru.romanow.emptyauthorization; // OK
```

```java
package ru.romanow.empty_authorization; // BAD
```

### Правила именования классов, интерфейсов

Именование классов и интерфейсов должно быть в UpperCamelCase.

Интерфейс может называться словосочетанием (например, `UserService`, `DataPreparationService`) или
наречием (`Closeable`, `Iterable`). Именование наречия означает, что интерфейс маркировочный, т.е.
класс является чем-то (например `Cloneable` – имеет методы для клонирования, `Iterable` – можно
использовать в цикле`for`).

В именовании классов (и интерфейсов, если применимо) можно указывать суффикс:

* `Service` – для `@Service`;
  ```java
  @Service
  public class UserServiceImpl
        extends UserService {}
  ```
* `Controller` – для `@RestController`, `@Controller`;
  ```java
  @RestController
  public class UserController {}
  ```
* `Controller` или `Advice` – для `@RestControllerAdvice`, `@ControllerAdvice`;
  ```java
  @RestControllerAdvice
  public class ExceptionController {}
  ```
* `Repository` - для наследников от `CrudRepository`, `JpaRepository` и т.д.;
  ```java
  public class UserRepository
        extends JpaRepository<User, Long> {}
  ```
* `Dao` – для классов доступа к данным, работающим с SQL или `EntityManager`, а так же помеченных (
  но не
  обязательно) `@Repository`.
  ```java
  @Repository
  public class UserDaoImpl
      extends UserDao {

      @PersistenceContext
      private final EntityManager entityManager;
  }
  ```
* Исключение: если класс является расширением
  `JpaRepository` ([Custom Implementations for Spring Data Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.custom-implementations)),
  то остается суффикс `Repository`.
* Аннотация `@Component` является базовой и выносить ее в название класса не нужно.
* Если интерфейс называется словосочетанием, то, если реализация единственная, добавляем суффикс
  `Impl`. Если есть несколько наследников, то каждый наследник называется в соответствии с решаемой
  задачей:
    * `UserService` -> `UserServiceImpl`;
    * `DataPreparationService` -> `XmlDataPreparationService`, `CsvDataPreparationService`.
* Маркировочные интерфейсы в именовании классов не участвуют.

### Критерии использования inner классов

Inner классы используем только в случае, если описываемый объект является составной частью общего.
Например:

* составной ключ в сущности `@EmbeddedId`;
* частью конфигурации, например profile-specific настройки или описание контекста в тестах;
* внутренним объектом в сервисе, используемым _внутри_ сервиса для передачи большого количества
  параметров в методы.

Если модели являются проекцией сущностей в базе данных, то они не могут быть вложенными.

### Именование веток

Все commits должны ссылаться на задачу в Jira. Можно настроить `Tasks & Context` (`Tools` ->
`Tasks & Context`-> `Configure Servers` -> `Jira`) и управлять задачами через Intellij Idea. Так же
там создаются
`Changelist` с именем задачи и при commit автоматически подставляется это описание.

Ветки называть в формате Gitflow: (`feature` | `bugfix`)/`<task-number>`. Опционально, пояснение о
решаемой задаче.

Если в рамках ветки решается несколько связанных задач, то можно называть ветку по имени главной
задачи (User Story, Technical Task).

### Unit тесты

Обязательно пишем тесты на сервисы (бизнес логика). Эти тесты стараемся делать простыми без
использования `@SpringBootTest` и его производных `@DataJpaTest`, `@WebMvcTest`.

Если требуется использование `@SpringBootTest`, то в самом тесте в inner классе описываем контекст,
а в `@SpringBootTest` в `classes` указываем ссылку на этот контекст.

* Тесты не должны быть зависимыми друг от друга или содержать глобальное состояние.
* Тесты должны тестировать только один класс, остальные зависимости заменять `mock`. Писать сквозные
  тесты плохо, т.к. это усложняет их поддержку.

В тестах используем [Mockito](https://site.mockito.org/) для создания заглушек
и [Assertj](https://assertj.github.io/doc/) для проверки результата.

```java

@ExtendWith(MockitoExtension.class)
class PersonServiceTest {

    private static final int DEPARTMENT_ID = 200;
    private static final int PERSON_ID = 100;

    private PersonDao person;
    private PersonService personService;

    @BeforeEach
    void init() {
        personDao = Mockito.mock(PersonDao.class);
        personService = new PersonServiceImpl(personDao);
    }

    @Test
    void when_findById_then_success() {
        // Given
        final Person person = buildPerson(PERSON_ID);
        final Department department = person.getDepartment();
        when(personDao.findById(PERSON_ID)).thenReturn(person);

        // When
        final PersonFullResponse personResponse = personService.getById(PERSON_ID);

        // Then
        assertThat(personResponse.getId()).isEqualTo(PERSON_ID);
        assertThat(personResponse.getFullName()).isEqualTo(
                person.getLastName() + " " + person.getFirstName() + " " + person.getMiddleName());
        assertThat(personResponse.getAge()).isEqualTo(person.getAge());

        final DepartmentShortResponse departmentResponse = personResponse.getDepartment();
        assertThat(departmentResponse.getId()).isEqualTo(department.getId());
        assertThat(departmentResponse.getName()).isEqualTo(department.getName());
    }

}
```

Если метод не возвращает значения, то можно проверять его работоспособность по косвенным признакам,
например через `verify`.

При написании unit тестов разделяем тест на три части:

* `given` – подготовка данных, создание `mock`, `spy` и задание желаемого поведения для них;
* `when` – выполнение тестируемой операции;
* `then` – проверка результата.

Тесты именуем в соответствии с конвенцией: `when_<method>_then_<expected-result>`, например:

```java

@ExtendWith(MockitoExtension.class)
class ProductScenarioServiceTest {

    @Test
    void when_findProductScenarioByUid_then_success() {
    }

    @Test
    void when_findProductScenarioByUid_then_notFoundException() {
    }

}
```

Чтобы подробнее описать что делает этот тест, можно использовать `@DisplayName`.

### Логирование

Сообщения в логи пишем _на английском_ языке.

Из сообщения логов должно быть понятно, с какой сущностью выполнялась операция или какая возникла
ошибка. Если речь о сущностях, то выводить ее тип и `uid`, для классов моделей - `toString()` или
более короткую информацию.

```java

@Slf4j
class Main {

    public static void main(String[] args) {
        log.debug("Create new Process {}", process);
        log.info("Check process status uid '{}' and startDate '{}'", processUid, startDate);
    }

}
```

Для добавления параметров в логи используем `{}`, конкатенацию использовать не нужно.

Если при записи в лог объекты подготавливаются, например `List<Process>` -> `List<UUID>`,
то эту запись заворачиваем в условие проверки включения уровня логгирования.

```java

@Slf4j
class Main {

    public static void main(String[] args) {
        if (log.isDebugEnabled()) {
            log.debug("Remove temporary data for processUids: [{}]",
                      processes.stream().map(c -> c.getUid()).toList());
        }
    }

}

```

По ходу выполнения каждой бизнес операции пишем логи:

* `debug` – информация позволяющая разбираться в ошибках. В проблемных местах подробное описание,
  что вызывается и с какими параметрами. Логирование целого запроса только на этом уровне. Если
  требуется вывести полную информацию об объекте, то переопределить для него `toString()` или
  реализовать метод, который будет возвращать нужную информацию об объекте (например, если в объекте
  существуют поля, которые мы не можем светить в логах).
* `info` – информация о ходе выполнения бизнес-процесса, перехода со стадии на стадию.
* `warn` – ошибки или exception, которые не влияют фатально на ход выполнения (например,
  `NumberFormatException` при разборе `String`, если поле может быть `null`).
* `error` – exception, при возникновении которых, невозможно продолжать выполнение операций.

Для логгирования запросов / ответов используем [logbook](https://github.com/zalando/logbook).

### Использование профилей

Разделение на профили нужно для прозрачности настройки приложения в разных средах. Разбиение по
профилям выполняется только на уровне профиля приложения, используются стандартные механизмы Spring.

Для каждого контура (`dev`, `stage`, `prod`) описывается свой собственный профиль. Credential к
базе данных, сертификаты к Kafka и остальные важные настройки будут задаваться через переменные
среды, которые будут браться из Secret и ConfigMap.

### В OpenAPI поддерживать консистентность по ошибкам в `@ApiResponse`

Используем OpenAPI версии 3, реализация [`springdoc`](https://springdoc.org/).

При добавлении новых ошибок при обработке вызова, они должны быть отображены в `@ApiResponse`. Это
нужно, чтобы декларация метода в OpenAPI была консистентна с кодом, т.е., чтобы при вызове метода не
было не описанных кодов ответов (кроме 500).

### Использование `WebClient`

Используем `RestClient` или `WebClient`, причем создаем через `WebClient.Builder`.

Работу с `WebClient` убираем в отдельны сервис, который инкапсулирует в себе всю логику сборки
запроса, вызова и обработки результатов запроса. Сервис имеет суффикс `Client`.

### Использование готовых библиотек, а не разработка своих решений

Для решения типовых задач ищем и используем готовые библиотеки с высоким рейтингом. Например, для
проверки наличия
непустых символов в строке можно написать свой метод `hasChars`:

```java
public boolean hasChars(@Nullable String str) {
    return str != null && str.chars().anyMatch(c -> c != 0);
}
```

Но на этот метод нужно написать тесты и проверить, что реализация оптимальна. Но лучше использовать
готовый метод `org.springframework.util.StringUtils.hasText(@Nullable String str)`. Т.к. библиотеку
Spring поддерживает community, реализация таких методов будет не хуже, чем самописная.

Аналогичная ситуация с самописными реализациями:

* транслитерация rus -> latin: [Iuliia](https://github.com/Homyakin/iuliia-java);
* сборка SQL для записи скрипта в файл: [SqlBuilder](https://openhms.sourceforge.io/sqlbuilder/).

### Использование валидации методов JSR303

Если требуется проверить аргументы метода в контроллере, то на контроллер ставится аннотация
`@Validated` и на аргументы ставятся необходимые аннотации `@NotEmpty`, `@Positive`, `@Pattern` и
т.д. Аннотации `@NotNull` в аргументах бессмысленны, т.к. `@RequestParam`, `@QueryParam` имеет
параметр `required = true` по-умолчанию, т.е. при вызове метода эти аргументы уже точно не `null`.

Если аргумент должен иметь определенный тип (например `Double`, `UUID` и т.д.), то нужно сразу
ставить целевой этот тип данных, а не принимать все в `String`, а потом паттернами проверять
соответствие типу. Т.к. наше API для внутреннего использования, мы можем сразу указывать необходимый
тип данных, и в случае несоответствия аргумента типу просто возвращать 500 ошибку.

```java

@Validated
@RestController
@RequestMapping("/api/v1/process")
public class ProcessController {

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public List<ProcessResponse> search(
            @RequestParam @NotEmpty String name,
            @RequestParam @Future UUID data
    ) {
    }

}
```

Для валидации полей в объекте, в контроллере валидируемый объект помечается аннотацией `@Valid`, а в
самом объекте аннотации ставятся над полями.

```java

@RestController
@RequestMapping("/api/v1/process")
public class ProcessController {

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEnity<Void> create(@Valid @RequestBody ProcessCreateRequest request) {
    }

    @Data
    @Accessors(chain = true)
    public static class ProcessCreateRequest {

        @NotEmpty
        private String name;

        private String description;

        @NotNull
        @Size(min = 0, max = 100)
        private Integer percentage;

    }

}
```

### Переводы сообщений валидации

Аннотации валидации JSR-303 имеют параметр `message`, который ссылается на
`ValidationMessages.properties` внутри модуля `org.hibernate.validator:hibernate-validator`.

Для описания своих сообщений валидации, создаем файл
`resources/ValidationMessages_<locale>.properties` и в нем описываем коды и переводы по аналогии с
`messages.properties`.

При использовании аннотаций указываем параметр `@NotEmpty(message = "{field.not.empty}")` со ссылкой
на коды. Важно заметить, что код должен быть указан в фигурных скобках, без этого `message` будет
воспринят как готовый перевод. В сообщениях валидации можно использовать аргументы из аннотации
валидации.

```properties
# ValidationMessages_en.properties
field.not.empty=Field must be not empty
field.fixed.size=Field must be between interval {min} to {max}
```

```java

@Data
@Accessors(chain = true)
public class ObjectCreateRequest {

    @NotEmpty(message = "{field.not.empty}")
    private String name;

    @NotNull
    @Size(min = 0, max = 100, message = "{field.fixed.size}")
    private Integer percentage;

}
```

### Обработка exceptions

Если нужно сделать переводы сообщений, то можно воспользоваться стандартным механизмом Spring i10n.

Создается файл `resources/messages_<locale>.properties`, где locale – сокращенное (`ru`, `en`) или
полное имя (`ru_RU`, `en_US`) локали.

```properties
# messages_ru.properties
product.code.not.found=Код продукта не найден
```

```properties
# messages.properties
product.code.not.found=Product code not found
```

Создаем сервис `MessageHelper`, если `Locale` не передается при вызове метода, то
используется `LocaleContextHolder.getLocale()`, который берет Locale из параметров запроса.

```java

@Component
public class MessageHelper {

    private static MessageSource messageSource;

    public MessageHelper(MessageSource messageSource) {
        MessageHelper.messageSource = messageSource;
    }

    @NotNull
    public static String tr(@NotNull String code) {
        return tr(code, null, LocaleContextHolder.getLocale());
    }

    @NotNull
    public static String tr(@NotNull String code, @NotNull Object[] args) {
        return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
    }

}
```

## Работа с JPA и транзакциями

Подробнее [jpa-example](https://github.com/Romanow/jpa-example/).

### Использование `@Transacational`

Если в рамках запроса выполняется модификация нескольких таблиц, то без использования общей
транзакции в случае ошибки откат изменений не будет выполнен или будет выполнен частично, что
приведет к неконсистентности данных.

Транзакции нужно использовать на уровне service, т.к. именно там находится бизнес-логика приложения
и именно этот слой ответственен за корректность (консистентность) работы с данными.

Если в рамках бизнес операции используются только запросы на чтение, то нужно в транзакции указать
`@Transactional(readOnly = true)`.

Аннотацию `@Transactional` нужно указывать в реализации и лучше аннотировать ей каждый метод, где
это нужно. Помечать`@Transactional` декларацию методов в интерфейсе не стоит, т.к. это выдает детали
внутренней реализации и, если в реализации этой аннотации не будет, то по факту транзакция
создастся (т.к. Spring увидит `@Transactional` в интерфейсе), но по коду это будет неочевидно.

При использовании JPA репозиториев транзакция не создается из воздуха, методы интерфейса
делегируются классу `SimpleJpaRepository`, который помечен аннотацией
`@Transactional(readOnly = true)`.

### Выключение параметра `spring.jpa.open-in-view=false`

Если включен параметр `spring.jpa.open-in-view`, то класс `OpenEntityManagerInViewInterceptor` в
методе `preHandle` открывает `EntityManager` для текущего запроса, т.е. Spring создает обрамляющую
транзакцию на весь запрос.

Это очень плохой подход, т.к. он может привести к утечке соединений и падению приложения. Из-за
вопросов совместимости, в Spring Boot 3.x этот параметр остался включен. Настройка
`spring.jpa.open-in-view` всегда должна быть выключена в явном виде, это значит, что в коде нужно
использовать ручное управление транзакциями.

### Разбиение по доменным сущностям

Делать подпакеты можно, например request / response в `models`, но если требуются подпакеты в
`service`, обычно это значит что требуется разбиение на более мелкие доменные области.

Доменная область обычно 1 к 1 связана с бизнес-процессом, т.е. у вас в одном пакете есть
контроллеры (и сервисы), которые выполняют разную функциональность из разных Use Case (работа с
пользователем (создание, блокировка) и работа с кошельком (создание, пополнение, закрытие)), то это
обычно различные доменные области.

Если у вас есть необходимость в sql / jpa запросе использовать join на таблицы из разных доменных
областей, то лучше это делать в java коде, потому что в случае возможного распила сервиса на части,
части этого запроса могут относиться к разным доменным областям, а значит join придется распиливать.

К одной доменной сущности могут относиться объекты, которые будут невалидны без основной сущности.
Например, `User` ->`Address`, адрес будет невалиден без привязки к пользователю, но `Address` ->
`Country`,`Address` -> `City` уже не будут в одном домене, т.к. `Country` и `City` могут
потребоваться в других процессах.

### Если в `JpaRepository` метод короткий, то его можно использовать без `@Query`

Если условие поиска содержит одно или два поля, или одно поле и условие сортировки, то такой запрос
можно описать словами без `@Query`.

```java
public interface UserRepository
        extends JpaRepository<User, Long> {

    Optional<User> findByUid(UUID uid);

    List<User> findByStatusOrderByCreatedDate(String status);

    List<User> findByFirstNameAndLastName(String firstName, String lastName);

    @Query("select u from User u where u.status = :status and u.firstName = :firstName order by u.createdDate asc")
    List<User> findByStatusAndName(
            @Param("status") String status,
            @Param("firstName") String firstName
    );

}
```

Запросы, содержащие больше одного условия в блоке `where` или дополнительные условия `order by`,
`group by`, `having`, то для улучшения читабельности, запрос нужно записывать в несколько строк,
каждый блок на новой строке.

Если требуется делать запросы, где условия поиска зависят от входных параметров, то использовать
либо `Example<T>`, либо репозиторий наследовать от `JpaSpecificationExecutor<T>` и работать с
Criteria API в `findAll(Specification<T> spec)`.

### Описание `@Entity`

* При описании `@Entity` данных всегда описываем `@Table` с именем таблицы в базе данных.
* Все описания колонок в `@Entity` должны максимально соответствовать колонкам в базе данных. При
  описании колонки в аннотации `@Column` всегда указывать `name` и дополнительные аргументы,
  например `nullable`, `length` (для `VARCHAR`), `scale` и `presicion` (для `NUMBER`). Если для
  колонки задается `DEFAULT`, то он описывается в
  `columnDefinition = "BOOLEAN NOT NULL DEFAULT false"`.
* Если объект имеет уникальное поле (например `uid`), то `equals` и `hashCode` нужно делать от него.
  Не нужно делать `equals` и `hashCode` от `@Id` (если это поле генерируется Hibernate на этапе
  сохранения).
* Если `equals`, `hashCode`, `toString` реализуется для `@Entity`, то в них не должно быть связных
  сущностей (даже если они `fetch = EAGER`).
* В `@Entity` можно использовать аннотации `@Getter`, `@Setter`, `@Accessors(chain = true)`.
* Если `@Entity` ссылается на другую сущность, то в `@JoinColumn` всегда описываем имя колонки
  и имя constraint.
  ```java
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "process_id", foreignKey = @ForeignKey(name = "fk_process_version_process_id"))
  private Process process;
  ```

### Правила работы с `@OneToMany`, `@ManyToOne`, `@ManyToMany`

* По-умолчанию для `@OneToMany`, `@ManyToOne`, `@ManyToMany` используем тип загрузки
  `FetchType.LAZY`.
  Оставлять `FetchType.EAGER` в `@ManyToOne` можно только можно в случае, если основная сущность
  всегда нужна в связке с дочерней.
* Для связи `@OneToMany` `LAZY` по дефолту, его прописывать не нужно.
* Для связи `@OneToOne` реализовать `LAZY` без костылей нельзя. Но обычно использование `@OneToOne`
  означает, что дочерняя сущность является частью общей сущности, что не противоречит описанному
  выше
  правилу.
* Изменение типа связи с `LAZY` на `EAGER` крайне не рекомендуется, это может очень негативно
  сказаться на производительности, т.к. при поднятии одной сущности, будут подниматься еще N
  дополнительных сущностей.
* При этом, если связь помечена `LAZY`, а обращение к ней выполняется вне транзакции, то будет
  выброшен `LazyInitializationException`. Для предотвращения такой ситуации нужно явно использовать
  транзакции и (или) использовать `join fetch` и `@EntityGraph` в случае, когда эти данные нужны в
  получаемом результате.
* Аннотация `@JoinColumn` может быть по обе стороны связи (`@OneToMany`, `@ManyToOne`), она
  определяет главную сущность в отношении.

### JPA Auditing

Если требуется использовать поля: дата создания, дата последней модификации, имя пользователя,
который создал или изменил запись, то нужно использовать стандартные средства JPA Auditing с помощью
аннотаций:

* `@CreatedDate`, `@LastModifiedDate`
* `@CreatedBy`, `@LastModifiedBy`

Для их использования нужно создать `DateTimeProvider` и в самом `@Entity`
прописать использование `@EntityListeners(AuditingEntityListener.class)`

### Форматирование SQL

* Зарезервированные слова _SQL_ пишутся с большой буквы.
* Зарезервированные слова в _HQL_ пишутся с маленькой буквы.
* Имена таблиц и полей в таблице пишутся в snake_case.
* При создании таблицы ограничения `CHECK`, `CONSTRAINT` по возможности описываются вместе с полем,
  к которому относятся.
* Все названия специальных объектов должны иметь имена: `<prefix>_<table-name>_<field-name>`.

```sql
CREATE SEQUENCE seq_process_order AS BIGINT START WITH 1 INCREMENT BY 10;

CREATE TABLE process
(
    id          BIGSERIAL PRIMARY KEY,
    uid         UUID         NOT NULL,
    name        VARCHAR(255) NOT NULL,
    description VARCHAR(255),
    "order"     INT          NOT NULL DEFAULT NEXTVAL('seq_process_priority'),
    priority    INT          NOT NULL CHECK (priority >= 0 AND priority < 10)
);

CREATE INDEX idx_process_name ON process (name);
CREATE UNIQUE INDEX ui_process_uid ON process (uid);

CREATE TABLE process_status_history
(
    id         BIGSERIAL PRIMARY KEY,
    status     VARCHAR(80),
    process_id INT
        CONSTRAINT fk_process_status_history_process_id
            REFERENCES process (id) ON DELETE CASCADE
);

CREATE INDEX idx_process_status_history_process_id ON process_status_history (process_id);
```

Префиксы объектов:

* `index` – `idx_`;
* `unique index` – `ux_`;
* `sequence` – `seq_`;
* `foreign key` – `fk_`.

Для всех foreign key создаем индексы.

В реляционных базах данных Primary Key и Foreign Key используются для контроля целостности данных.
Это позволяет блокировать удаление строки, если на нее ссылаются другие записи.

Менять дефолтное поведение `ON UPDATE` / `ON DELETE` на `CASCADE` нужно делать осмысленно, т.к.
можно неявно грохнуть половину базы данных.

Для Primary Key
индекс [создается по-умолчанию](https://www.postgresql.org/docs/current/indexes-unique.html). Для
Foreign Key обязательно создавать индекс, иначе `JOIN` между таблицами будет выполняться как
`Sequence Scan`.

### Использование UUID для взаимодействия между сервисами

Если требуется создать связь сущностей между сервисами, то _дополнительно_ к `private key` создается
поле `uid UUID NOT NULL`, которое является внешним ключом между базами данных.

Т.е. если требуется вывести какую-то информацию о сущности на UI или сделать ссылку на нее из
другого сервиса, то в сущности заводится поле uid, и все взаимодействие идет через него. Внутренние
`primary key` используются только для ссылок _внутри одной_ базы данных.

Т.к. uid должен быть уникальным ключом, на него создается `unique index`:

```sql
CREATE TABLE process
(
    id  BIGSERIAL PRIMARY KEY,
    uid UUID NOT NULL
);

CREATE UNIQUE INDEX ux_process_uid ON process (uid);
```

### Миграции в `liquibase`

Для миграций используем
[`liquibase`](https://docs.liquibase.com/concepts/introduction-to-liquibase.html).

Именование файлов миграции `YYYYMMDD_HHMI_<task-number>.xml`: `20211231_2359_JIRA-1000.xml`

Имена шагов миграций внутри `liquibase` состоит из `<NN>-<описание-изменений>`.

Для увеличения надежности используем:

* `preCondition` – проверка того, что скрипт нужно выполнять. Это очень полезный инструмент, т.к. он
  предотвращает повторное создание уже существующих объектов.
* `rollback` – скрипт отката, нужен по двум причинам:
    * если изменения были применены по-ошибке или содержат ошибку;
    * при работе тестировщиков на тестовом контуре при взятии в работу задачи с другим набором
      миграций.

Комментарии к простым миграциям не нужны. Если скрипт сложный или объемный (более 1 экрана), то
комментарий обязателен.

Используем описание миграций в xml, если выполняется изменение DDL. Если требуется модификация
данных, то используем SQL.

```xml

<changeSet id="01-update-currency-code-for-rubles" author="<автор>">
    <preConditions onFail="MARK_RUN" expectedResult="1">
        SELECT 1 FROM currency WHERE id = 'RUB' and name='Российский рубль'
    </preConditions>

    <comment>Проставляем правильное имя для RUB</comment>
    <sql splitStatements="false">
        UPDATE currency SET name='Российский рубль' WHERE id = 'RUB'
    </sql>

    <rollback>
        <sql>
            UPDATE currency SET name='Деревянный' WHERE id = 'RUB'
        </sql>
    </rollback>
</changeSet>
```

### Использование MapStruct

Если в проекте требуется делать преобразования `entity` -> `response model`, `request model` ->
`entity`, то нужно использовать
[`MapStruct`](https://mapstruct.org/documentation/stable/reference/html/).

Задача мапперов просто переложить готовые данные из одного объекта в другой. Мапперы следует делать
быть максимально простыми, в идеальном случае они должны заниматься только перекладыванием плоских
полей (`String`, `Integer`, `BigDecimal`) из объекта в объект.

Если объект является составным, то для каждой доменной сущности должен быть написан свой mapper.
Структрурная зависимость мапперов должна повторять структуру доменной области для простоты
понимания, поддержки и соблюдения Single Responsibility.

Для сложных составных объектов нужно разделять операции создания и редактирования.

* Для связи мапперов использовать `uses`: `@Mapper(uses = { AddressMapper.class }`.
* Если ваше приложение написано на Spring Boot, то мапперы тоже должны быть под управлением
  Spring: `@Mapper(componentModel = "spring")`.
* Для создания мапперов использовать
  `@Mapper(componentModel = "spring", injectionStrategy = InjectionStrategy.CONSTRUCTOR)`, что
  позволит создавать маппер в тестах без создания контекста Spring.
* Нужно избегать внедрения других сервисов в мапперы, это упростит поддержку. Лучше сделать что
  маппер сможет, а остальное уже доставить в сервисе вызова и туда внедрить зависимости.
* Не рекомендуется использовать `@BeforeMapping`, так как входная (source) сущность может находиться
  в `Persistence Context` и её изменения в рамках метода `@BeforeMapping` могут быть неявно
  сохранены.
* Не использовать `expression="java()"` для методов с бизнес-логикой – это очень сложно тестировать.
* Полезно использовать `@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR)`, а все неиспользуемые
  поля в mapping явно указывать через `@Mapper(ignore = true, target = "..."")`. Это позволит не
  пропустить поля в итоговом объекте.
