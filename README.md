# Code Conventions

## Code Style

За основу разметки в коде берем [Google Java Code Style](https://google.github.io/styleguide/javaguide.html).

Описание [Code Style](.idea/codeStyles/Project.xml)
кладем в папку `.idea/codeStyle/Project.xml` и там же рядом создаем
файл [`codeStyleConfig.xml`](.idea/codeStyles/codeStyleConfig.xml), чтобы Code Style использовался из проекта:

```xml

<component name="ProjectCodeStyleConfiguration">
    <state>
        <option name="USE_PER_PROJECT_SETTINGS" value="true"/>
    </state>
</component>
 ```

Папку `.idea` убираем из `.gitignore` добавляем в индекс. В папке `.idea` есть свой [`.gitignore`](.idea/.gitignore),
где прописано, что user-specific файлы не попадали в git.

## Соглашение по коду

#### Использование `final` для локальных переменных

Для локальных переменных используем модификатор `final` где это возможно, это улучшает performance, т.к. оптимизатор
понимает, что значение переменной присваивается один раз и меняться не будет.
[The Java final Keyword – Impact on Performance](https://www.baeldung.com/java-final-performance)

#### Количество параметров в методе

Если в методе больше 5 параметров, то их нужно оборачивать в объект.

Большое количество параметров может свидетельствовать о том, что:

* метод нарушает правило Single Responsibility – выполняет разные задачи.
* в случае методов в 1-2 строки и 5+ параметрами: метод слишком декомпозирован и его вызов лучше заменить inline-кодом.

#### Использование интерфейсов

Интерфейс – это контракт класса. Он должен содержать только методы, которые описывают его _зону ответственности_ (
Responsibility). Вынесение приватных методов на уровень интерфейса нарушает парадигму Инкапсуляции.

Интерфейс не должен содержать методы (`default` методы), крайне нежелательно описывать в нем константы.

Для всех классов под управлением Spring, которые помечены аннотацией `@Service`
и `@Repository` (т.е. слой бизнес логики и DAO через `@PersistenceContext`), использование _интерфейсов обязательно_.

Интерфейсы не нужно описывать для утильных классов, контроллеров (`@Controller`), репозиториев (наследников
`CrudRepository`) и т.п.

Если требуется описать какую-то общую логику, то для этого используется абстрактный класс.

#### Порядок описания методов

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

#### Использование Optional

`java.util.Optional` можно использовать как возвращаемое значение, это полезно когда требуется сделать chaining.

```jshelllanguage

    @Override
    @Transactional
    public Product findByUidOrDefault(@NotNull UUID uid) {
        return dictionaryClient.findByUid(uid)
                .map(ProductMapper::toModel)
                .orElseGet(() -> new Product("N/A"));
    }
```

Но для описания параметров метода его использовать не стоит, лучше в явном виде описать, что метод может
принимать `@Nullable`.

#### Использование префиксов в именах методов

Если метод имеет префикс `find` или `get`, то он возвращает значение:

* `find` – метод возвращает `Optional`, кроме случаев когда возвращается `Pageable`, `List`, `Map` и т.п. (они должны
  возвращать пустую коллекцию). Метод всегда помечается `@NotNull`. Если значение не найдено, то
  возвращается `Optional.empty()`.
* `get` – если значение не найдено, то выбрасывается исключение.

Аналогичное поведение в методах `getById` в `JpaRepository` и `findById` в `CrudRepository`.

Если метод имеет префикс `build` – то обычно это некоторый утильный метод для сборки объекта из других объектов.

Методы, имеющие префикс `get`, `find`, `build` никогда не могут вносить изменения в базу данных. Если это методы на
уровне сервисов, то они должны иметь аннотацию `@Transactional(readOnly = true)` в случае, если они выполняют обращение
к базе данных.

Такое именование метода запрещено (!!!):

```jshelllanguage

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

#### Использование `@Nullable`, `@NotNull`, `@Contract`

В сервисных методах описываем параметры и возвращаемые значения с помощью аннотаций `@NotNull` и `@Nullable` (из
пакета `org.jetbrains:annotations`). Задача использования этих аннотация в статическом анализе NPE с помощью Intellij
Idea. Они имеют `Retention Policy = CLASS` и недоступны в runtime, в отличие от lombok `@NotNull`, они не
выбрасывают `NullPointerException` в runtime.

Для более сложного инструктирования статического анализатора можно использовать
аннотацию [`@Contract`](https://www.jetbrains.com/help/idea/contract-annotations.html).

#### Общие требования

1. Если Intellij Idea подсвечивает код, то его нужно исправить.
2. Переопределенные методы всегда помечаем аннотацией `@Override`.
3. Можно использовать `var` для описания локальных переменных, если тип переменной очевиден из правого выражения. В
   цикле всегда пишем полный тип.
   ```jshelllanguage
   var users = List.of("Alex", "Andrew", "Max", "Anton", "Gleb");
   for (final String user: users) {
       System.out.println(user);
   }
   ```
4. Если `var` не используется, то используем `diamond operator <>`
   ```jshelllanguage
   private Map<String, Pair<Integer, String>> ratingByUsers = new HashMap<>();
   ``` 
5. Для создания список используем `List.of()`, для создания map `Map.of()`. При этом создаются readonly коллекции.
   ```jshelllanguage
   private List<String> users = List.of("Alex", "Andrew", "Max", "Anton", "Gleb");
   private Map<String, Integer> ratingByUsers = Map.of(
       "Alex", 1,
       "Max", 2,
       "Andrew", 3
   );
   ```
6. Не используем checked-исключения, только наследников от `RuntimeException`.
7. Кране нежелательно перехватывать обобщенные исключения `Throwable`.
8. Никогда нельзя перехватывать исключения типа `Error`.
9. Использовать пробелы вместо табуляции.
10. Не использовать в заголовке класса `@author` (отключить в Intellij Idea). Они указывают кто создал файл, но после
    этого много народу могут менять этот файл. Для понимания, кто автор изменений в файле, использовать `git annotate`.
11. Метод не должен занимать больше 3 экранов. Если метод больше, то следует разносить его на более мелкие методы.
12. Все magic numbers, constant strings выносим в именованные константы. Константа
    описывается `(public|private) static final`. Так же можно для набора констант использовать enum.
13. Не использовать XML Dom, XPath для навигации по объектам.
14. Нельзя использовать хранимые процедуры.
15. Строить бизнес-логику на exception можно, например, если в процессе выполнения были получены какие-то данные, работа
    с которыми невозможна.
16. Если есть параметризованные сложные `SQL`, то для их построения использовать `QueryDSL` или `Criteria API`, не
    клеить их руками через конкатенацию.
17. Контролеры максимально простые, вся бизнес логика внутри сервисов. Объекты `HttpServletRequest`,
    `HttpServletResponse`, `SecurityContextHolder` и т.п. могут использоваться только в слое web, а в сервисы
    пробрасываются уже результат, который мы их них получили. В противном случае мы мешаем логику представления с
    бизнес-логикой.
18. Если проверка валидации требует доступ к базе данных, то ее нужно переносить в слой сервисов, иначе реализовывать
    через аннотации JSR303 или кодом в контроллере.
19. Аннотацию `@SneakyThrows` можно использовать в тестах в любом месте.
20. В коде можно использовать аннотацию `@SneakyThrows` только для _приватных методов_. `@SneakyThrows` нельзя
    использовать в методах, которые оборачиваются другими аннотациями Spring (например, `@Transactional`), потому что
    она маскирует checked exception, делая возможным не описывать его в сигнатуре метода `throws`, Spring создает
    Dynamic Proxy ожидая корректную сигнатуру метода. [Объяснение](https://stackoverflow.com/a/56508336) <br/>
    Если требуется использовать `@SnekyThrows`, то выносим код в private метод:
    ```jshelllanguage
    @SneakyThrows
    private void deleteFile(@NotNull String path) {
        Files.deleteIfExists(path);
    }
    ```
21. Не использовать `printStackTrace()`, т.к. он не попадает в лог (ELK).
    ```jshelllanguage
    logger.error("", e);
    ```

#### Комментарии к коду

Комментарии пишутся с большой буквы.

Javadoc комментарии на каждый метод не нужны, они не несут полезной нагрузки и в случае изменения поведения метода,
можно забыть их исправить, что приведет негативному эффекту: описание метода не соответствует реализации.

Очевидные комментарии по коду тоже не нужны, в них нет никакого смысла:

```jshelllanguage
    // Ищем productScenario по calculationUid
    final List<ProductScenario> ps = productScenarioRepository.findByCalculationUid(calculationUid);
    // Цикл по списку productScenario
    for (var ps : productScenarios) {
        if (ps.status == ACTIVE) {
        ...
        }
    }
```

Название метода должно отражать что делает этот метод, при этом название должно оставаться
лаконичным: `createProductScenario`, `updateCalculation`, `generateExcel` и т.п. Если в названии нельзя однозначно
описать, что происходит в методе, то скорее всего нарушается принцип Single Responsibility.

Подробнее про названия описано в блоке `Названия методов`.

Если требуется закомментировать код, то нужно описать `TODO` с описанием что делает этот код и четкими критериями когда
этот он потребуется. Желательно прикрепить задачу.

Если в коде есть закомментированный код, которому больше 3 недель и к нему не описан `TODO`, то этот код можно удалять.

Комментарии обязательно оставлять к сложным кускам кода и неочевидным решениям, например:

```java

@EnableWebMvc
@Configuration
@Profile({"local", "docker"})
public class WebConfigurationLocal
        implements WebMvcConfigurer {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // Задаем message converters для того, чтобы в MappingJackson2HttpMessageConverter использовался
        // objectMapper, который настраивается Spring application.properties
        // StringHttpMessageConverter и ByteArrayHttpMessageConverter используются чтобы корректно отображался
        // OpenAPI endpoint, без них SwaggerWelcomeWebMvc :: openapiJson возвразает объект, обернутый в строку
        converters.add(new StringHttpMessageConverter());
        converters.add(new MappingJackson2HttpMessageConverter(objectMapper));
        converters.add(new ByteArrayHttpMessageConverter());
    }
}
```

Если бы здесь не было этого комментария, причины появления этого кода быстро забылись бы и появилось ощущение, что мы
без всякой нужны переопределяем существующее поведение. Удаление этого метода привело бы к тому, что настройка jackson
сериализатора в Spring Boot `spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS: false` (и другие) перестала бы
работать и даты стали возвращаться в виде timestamp.

#### Названия методов

Названия методов должно объяснять что он делает в общих чертах, т.е. определять его зону ответственности. Не нужно
пытаться запихнуть в название всю информацию о происходящих в нем действиях.

```jshelllanguage

    @NotNull
    @Override
    User createUser(@NotNull CreateUserRequest request) {
        ...
    }

    @Override
    void lockTransaction(@NotNull Transaction transaction) {
        ...
    }

```

#### Constructor Injection

Если класс под управлением Spring, то используем constructor injection.

Если в классе только один конструктор (отличный от конструктора по-умолчанию), то его не надо помечать `@Autowired`,
Spring поймет что используется constructor injection и будет пытаться подставить нужные beans.

Можно использовать аннотацию `@RequriedArgConstructor` и необходимые поля описывать как `private final`.

`@RequriedArgConstructor` не умеет работать с аннотациями на поля, т.е. если у нас в классе есть поля с `@Value`,
`@Qualifier` и т.п., то либо описываем их как обычные поля вне конструктора, либо создаем конструктор руками и там
описываем вне нужные аннотации.

`@AllArgConstructor` использовать не рекомендуется по описанным выше причинам.

```java

@Service
@AllArgsConstructor
public class MyEntityServiceImpl
        implements MyEntityService {
    private final MyEntityRepository myEntityRepository;
    private final MyEntityMapper myEntityMapper;
    private final RestTemplate restTemplate;

    @Value("${services.dictionary.url}")
    private String dictionaryUrl;
    
    ...
}
```

Использование constructor injection хороший вариант, потому что при написании unit тестов сразу понятно какие параметры
нужно передавать в метод. Так же, если мы видим, что для создания класса требуется передать большое количество
параметров в конструктор, то возможно класс занимается не только своей задачей и нарушает Single Responsibility.

#### Правила именования пакетов

Стараться использовать одно слово в именовании пакетов, если это невозможно, то используем lowerCamelCase или разделяем
точками (т.е. создавать подпакеты) или через, snake_case недопустим.

```java
package ru.vtb.calm.authorization; // OK
```

```java
package ru.vtb.calm.empty.authorization; // OK
```

```java
package ru.vtb.calm.emptyauthorization; // OK
```

```java
package ru.vtb.calm.empty_authorization; // BAD
```

#### Правила именования классов, интерфейсов

Именование классов и интерфейсов должно быть в UpperCamelCase.

Интерфейс может называться словосочетанием (например, `UserService`, `DataPreparationService`) или наречием (`Closeable`
, `Iterable`). Именование наречием означает, что интерфейс маркировочный, т.е. класс является чем-то (
например `Cloneable` – имеет методы для клонирования, `Iterable` – можно использовать в цикле `for`).

В именовании классов (и интерфейсов, если применимо) можно указывать суффикс:

* `Service` – для `@Service`;
  ```jshelllanguage
  @Service
  public class UserServiceImpl
        extends UserService {
    ...
  }
  ```

* `Controller` – для `@RestController`, `@Controller`;
  ```jshelllanguage
  @RestController
  public class UserController {
    ...
  }
  ```
* `Controller` или `Advice` – для `@RestControllerAdvice`, `@ControllerAdvice`;
  ```jshelllanguage
  @RestControllerAdvice
  public class ExceptionController {
    ...
  }
  ```
* `Repository` - для наследников от `CrudRepository`, `JpaRepository` и т.д.;
  ```jshelllanguage
  public class UserRepository 
        extends JpaRepository<User, Long> {}
  ```
* `Dao` – для классов доступа к данным, работающим с SQL или `EntityManager`, а так же помеченных (но не
  обязательно) `@Repository`.
  ```jshelllanguage
  @Repository
  public class UserDaoImpl
      extends UserDao {
  
      @PersistenceContext
      private final EntityManager entityManager;
      
      ...
  }
  ```
  Исключение: если класс является
  расширением `JpaRepository` ([Custom Implementations for Spring Data Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.custom-implementations))
  , то остается суффикс `Repository`.

Аннотация `@Component` является базовой и выносить ее в название класса не нужно.

Если интерфейс называется словосочетанием, то, если реализация единственная, добавляем суффикс `Impl`. Если есть
несколько наследников, то каждый наследник называется в соответствии с решаемой задачей.

Например:

* `UserService` -> `UserServiceImpl`;
* `DataPreparationService` -> `XmlDataPreparationService`, `CsvDataPreparationService`.

Маркировочные интерфейсы в именовании классов не участвуют.

#### Критерии использования inner классов

Inner классы используем только в случае, если описываемый объект является составной частью общего. Например:

* составной ключ в сущности `@EmbeddedId`;
* частью конфигурации, например profile-specific настройки или описание контекста в тестах;
* внутренним объектом в сервисе, используемым _внутри_ сервиса для передачи большого количества параметров в методы.

Если модели являются проекцией сущностей в базе данных, то они не могут быть вложенными.

#### Если в `JpaRepository` метод короткий, то его можно использовать без `@Query`

Если условие поиска содержит одно или два поля, или одно поле и условие сортировки, то такой запрос можно описать
словами без `@Query`.

```java
import java.util.UUID;

public interface UserRepository
        extends JpaRepository<User, Long> {

    Optional<User> findByUid(UUID uid);

    List<User> findByStatusOrderByCreatedDate(String status);

    List<User> findByFirstNameAndLastName(String firstName, String lastName);

    @Query("select u " +
            "from User u" +
            "where u.status = :status " +
            " and u.firstName = :firstName " +
            " order by u.createdDate asc")
    List<User> findByStatusAndName(@Param("status") String status, @Param("firstName") String firstName);
}
```

Запросы, содержащие больше одного условия в блоке `where` или дополнительные условия `order by`, `group by`, `having`,
то для улучшения читабельности, запрос нужно записывать в несколько строк, каждый блок на новой строке.

Использовать `nativeQuery` в репозиториях можно, но только когда запрос нельзя написать на HQL.

Если требуется делать запросы, где условия поиска зависят от входных параемтров, то использовать либо `Example<T>`, либо
репозиторий наследовать от `JpaSpecificationExecutor<T>` и работать с Criteria API в `findAll(Specification<T> spec)`.

#### Именование веток

Все commits должны ссылаться на задачу в Jira. Можно настроить `Tasks & Context` (`Tools` -> `Tasks & Context`
-> `Configure Servers` -> `Jira`) и управлять задачами через Intellij Idea. Так же там создаются `Changelist` с именем
задачи и при commit автоматически подставляется это описание.

Ветки называть в формате Gitflow: (`feature` | `bugfix`)/<Номер задачи>-<пояснение>. Суффикс названия с пояснением о
решаемой задаче очень желательно.

Если в рамках ветки решается несколько связанных задач, то можно называть ветку по имени главной задачи (User Story,
Technical Task).

#### `Given` - `When` - `Then` в структуре теста

При написании unit тестов разделяем тест на три части:

* `given` – подготовка данных, создание `mock`, `spy` и задание желаемого поведения для них;
* `when` – выполнение тестируемой операции;
* `then` – проверка результата.

Пример такого описания ниже в блоке _Unit тесты_.

#### Именование тестов

Тесты именуем в соответствии с конвенцией: `<method>_<expected-result>`, например:

```java

@ExtendWith(MockitoExtension.class)
class ProductScenarioServiceTest {

    @Test
    void findProductScenarioByUid_success() {
      ...
    }

    @Test
    void findProductScenarioByUid_notFound() {
      ...
    }
}
```

Чтобы подробнее описать что делает этот тест, можно использовать `@DisplayName`.

#### Unit тесты

Обязательно пишем тесты на сервисы (бизнес логика). Эти тесты стараемся делать простыми без
использования `@SpringBootTest` и его производных `@DataJpaTest`, `@WebMvcTest`.

Если требуется использование `@SpringBootTest`, то в самом тесте в inner классе описываем контекст, а
в `@SpringBootTest` в `classes` указываем ссылку на этот контекст.

* Тесты не должны быть зависимыми друг от друга или содержать глобальное состояние.
* Тесты должны тестировать только один класс, остальные зависимости заменять `mock`. Писать сквозные тесты плохо, т.к.
  это усложняет их поддержку.

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
    void findById_success() {
        // Given
        final Person person = buildPerson(PERSON_ID);
        final Department department = person.getDepartment();
        when(personDao.findById(PERSON_ID)).thenReturn(person);

        // When
        final PersonFullResponse personResponse = personService.getById(PERSON_ID);

        // Then
        assertThat(personResponse.getId()).isEqualTo(PERSON_ID);
        assertThat(personResponse.getFullName()).isEqualTo(person.getLastName() + " " + person.getFirstName() + " " + person.getMiddleName());
        assertThat(personResponse.getAge()).isEqualTo(person.getAge());

        final DepartmentShortResponse departmentResponse = personResponse.getDepartment();
        assertThat(departmentResponse.getId()).isEqualTo(department.getId());
        assertThat(departmentResponse.getName()).isEqualTo(department.getName());
    }
}
```

Если метод не возвращает значения, то можно проверять его работоспособность по косвенным признакам, например:

```kotlin
@Service
@Profile("local")
class LogNotificationService : NotificationService {
    private val logger = LoggerFactory.getLogger(LogNotificationService::class.java)

    override fun notify(event: PersonEvent) {
        logger.info(event.toString())
    }
}
```

```kotlin
@ExtendWith(MockitoExtension::class)
internal class LogNotificationServiceTest {

    @ParameterizedTest
    @MethodSource("factory")
    fun testNotify(event: PersonEvent) {
        mockStatic(LoggerFactory::class.java).use {
            val logger: Logger = mock(Logger::class.java)
            it.`when`<Logger> { LoggerFactory.getLogger(any(Class::class.java)) }
                    .thenReturn(logger)

            LogNotificationService().notify(event)
            verify(logger, times(1)).info("Event: ${event.eventType}, changed fields ${event.changedFields}")
        }
    }

    companion object {
        @JvmStatic
        fun factory(): Stream<PersonEvent> =
                Stream.of(PersonCreatedEvent(arrayOf()), PersonChangedEvent(arrayOf()), PersonRemovedEvent())
    }
}
```

Подробнее в статье [Unit Test](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=815987928).

#### Логирование

Сообщения в логи пишем _на английском_ языке.

Из сообщения логов должно быть понятно, с какой сущностью выполнялась операция или какая возникла ошибка. Если речь о
сущностях, то выводить ее тип и uid, для классов моделей - toString() или более короткую информацию.

```jshelllanguage
log.debug("Create new CalculationEntity {}", calculation);
    log.info("Start interpolation for calculationUid '{}' and factDataUid '{}'", calculationUid, factDataUid);
```

Для добавления параметров в логи используем `{}`, конкатенацию использовать не нужно.

Если при записи в лог объекты подготавливаются, например `List<CalculationEntity>` -> `List<UUID>`, то эту запись
заворачиваем в условие проверки включения уровня логгирования.

```jshelllanguage
if (log.isDebugEnabled()) {
    log.debug("Remove temporary data for calculationUids: [{}]",
            calculations.stream().map(c -> c.getUid()).collect(toList()));
}
```

По ходу выполнения каждой бизнес операции пишем логи:

* `debug` – информация позволяющая разбираться в ошибках. В проблемных местах подробное описание, что вызывается и с
  какими параметрами. Логирование целого запроса только на этом уровне. Если требуется вывести полную информацию об
  объекте, то переопределить для него `toString()` или реализовать свой метод, который будет выдавать нужную информацию
  об объекте (например, если в объекте существуют поля, которые мы не можем светить в логах).
* `info` – информация о ходе выполнения бизнес-процесса, перехода со стадии на стадию.
* `warn` – ошибки или exception’ы, которые не влияют фатально на ход выполнения (например, `NumberFormatException` при
  разборе `String`, если поле может быть `null`).
* `error` – exception, при возникновении которых, невозможно продолжать выполнение операций.

Для создания логгера используем аннотацию `@Slf4j`.

Для логгирования запросов используем [`logbook`](https://github.com/zalando/logbook).

```jshelllanguage

    @Bean
    public AbstractRequestLoggingFilter logFilter() {
        final AbstractRequestLoggingFilter filter = new AbstractRequestLoggingFilter() {
            @Override
            protected void beforeRequest(@Nonnull HttpServletRequest request, @Nonnull String message) {
            }

            @Override
            protected void afterRequest(@Nonnull HttpServletRequest request, @Nonnull String message) {
                final boolean contains = request.getServletPath().contains("/api");
                if (!contains) {
                    return;
                }
                log.info(message);
            }
        };
        filter.setIncludeQueryString(true);
        filter.setIncludePayload(true);
        filter.setMaxPayloadLength(50_000);
        filter.setIncludeHeaders(false);
        filter.setAfterMessagePrefix("REQUEST DATA: ");

        return filter;
    }
```

#### Правила описания моделей

* Для каждого endpoint описывать свои модели, разделяем модели для запросов и ответов, для запросов пишем
  суффикс `Request`.
* Модели (не `@Entity`), являющиеся проекциями доменных сущностей, именуются как сущности, но без суффикса `Entity`.
* Если модель не имеет прямое отношение к доменной сущности, то можно использовать суффиксы `Data`, `Info`, `Item`.
  Суффикс `DTO` не используем.
* Модели для запросов и ответов описываем отдельно, например:
  ```java
  @Data
  @Accessors(chain = true)
  public class CalculationCreateRequest {
      private String name;
      private String description;
      private Integer balanceSheetForecastingHorizon;
      private String type;
  }
  ```

  ```java
  @Data
  @Accessors(chain = true)
  public class Calculation {
      private UUID uid;
      private String name;
      private String description;
      private Integer balanceSheetForecastingHorizon;
      private String type;
  }
  ```
* Фильтрацию отображаемых rest-контроллером полей модели можно провести и с использованием `JsonView`, тогда не нужно
  будет создавать синтетические модели с их mappings.
* В именовании моделей можно использовать суффикс `List`, только если модель является наследником от коллекции. Это
  может потребоваться для корректного получения списка через `RestTemplate`.
* В модели допускается описывать несколько конструкторов, но для корректной сериализации всегда должен быть дефолтный
  конструктор.
* Поля с модификатором `final` нельзя использовать, т.к. это может привести к ошибкам сериализатора (если сериализатор
  использует при создании объекта дефолтный конструктор).
* В описании моделей всегда используем `Boolean`, т.к. если поле описано как `boolean` и в запросе приходит `null`, мы
  получаем `NullPointerException` с очень странным stacktrace.
* Если поле модели является коллекцией, то называем поле во множественном числе, нельзя использовать суффиксы `List`,
  `Collection` и т.п.
* `getter` и `setter` не должны содержать никакой логики (в том числе ленивая инициализация), т.к. модель является лишь
  представлением данных при передаче.
* В моделях допускается inline инициализация (или в конструкторе) сложных структур
  данных `private List<String> list = new ArrayList<>()`.
* Для сборки объектов использовать `@Builder`.
* В моделях можно использовать аннотацию `@Data` для создания `equals`, `hashCode`, `toString`. Если в эти методы
  требуется внести изменения (например убрать поле из `toString`), то переопределяем их руками. Использование
  аннотаций `@EqualsAndHashCode.Exclude`, `@ToString.Exclude` ухудшает читабельность.
* Для методов `equals`, `hashCode` и `toString` используем `EqualsBuilder`, `HashCodeBuilder` и `ToStringBuilder`
  соответственно из пакета Apache Commons Lang 3.

```java

@Data
@Accessor(chain = true)
public class Calculation {
    private UUID uid;
    private String name;
    private Integer version;
    private Boolean active;
    private List<CalculationVersion> calculationVersions;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        CalculationPeriodCreateRequest that = (CalculationPeriodCreateRequest) o;
        return new EqualsBuilder().append(uid, that.uid).isEquals();
    }

    @Override
    public int hashCode() {
        return new HashCodeBuilder(17, 37).append(uid).toHashCode();
    }

    @Override
    public String toString() {
        return new ToStringBuilder(this)
                .append("uid", uid)
                .append("name", name)
                .append("version", version)
                .append("active", active)
                .toString();
    }
}
```

#### Использование `enum`

TODO

## Архитектурные шаблоны

#### Использование `@Transacational`

Если в рамках запроса выполняется модификация нескольких таблиц, то без использования общей транзакции в случае ошибки
откат изменений не будет выполнен или будет выполнен частично, что приведет к неконсистентности данных.

Транзакции нужно использовать на уровне service, т.к. именно там находится бизнес-логика приложения и именно этот слой
ответственен за корректность (консистентность) работы с данными.

Если в рамках бизнес операции используются только запросы на чтение, то нужно в транзакции указать `@Transactional(
readOnly = true)`.

Аннотацию `@Transactional` нужно указывать в реализации и лучше аннотировать ей каждый метод, где это нужно. Помечать
`@Transactional` декларацию методов в интерфейсе не стоит, т.к. это выдает детали внутренней реализации и, если в
реализации этой аннотации не будет, то по факту транзакция создастся (т.к. Spring увидит `@Transactional` в интерфейсе),
но по коду это будет неочевидно.

При использовании JPA репозиториев транзакция не создается из воздуха, методы интерфейса делегируются
классу `SimpleJpaRepository`, который помечен аннотацией `@Transactional(readOnly = true)`.

```java

@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID>
        implements JpaRepositoryImplementation<T, ID> {
    ...
}
```

Подробнее в
статье [Использование библиотеки Hibernate](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=1140483682).

#### Выключение параметра `spring.jpa.open-in-view=false`

Если включен параметр `spring.jpa.open-in-view`, то класс `OpenEntityManagerInViewInterceptor` в методе preHandle
открывает EntityManager для текущего запроса, т.е. Spring создает обрамляющую транзакцию на весь запрос.

Это очень плохой подход, т.к. он может привести к утечке соединений и падению приложения. Из-за вопросов совместимости,
в Spring Boot 2.x этот параметр остался включен. Настройка `spring.jpa.open-in-view` всегда должна быть выключена в
явном виде, это значит, что в коде нужно использовать ручное управление транзакциями.

Подробнее в
статье [Использование библиотеки Hibernate](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=1140483682).

#### Общая структура проекта

![Project Structure](project_structure.png)

В корне проекта лежит главный файл`@SpringBootApplication`. Пакеты разбиваются по слоям:

* `web` – контроллеры и конвертеры;
    * `rest` – REST API.
    * `advice` – `@RestControllerAdvice` и прочие.
* `services` – бизнес логика и сервисы. Используем папку `impl` для разделения интерфейсов и реализаций и туда кладем
  package-private реализацию.
* `dao` – слой обращения к данным.
    * `entities` – доменные сущности, помеченные аннотацией `@Entity`.
    * `repositories` – DAO классы (c `@PersistanceContext`) и `JpaRepository`.
* `config` – `@Configuration` классы, каждая конфигурация по слоям выносится в отдельный класс. Т.е. bean и настройки,
  относящиеся к web, должно находится в WebConfiguration, но не обязательно только в нем одном. Можно разносить на более
  специфичные конфигурации. Главное не мешать конфигурации воедино, например конфигурация `@EnableJpaAuditing` не должна
  лежать в той же самой конфигурации, что и создание `RestTemplate`.
* `models` – классы моделей.
* `mappings` – mapstruct mappers, так же здесь могут находиться утильные сервисы для маппинга.
* `exceptions` – исключения.
* `utils` – разные утильные классы. Обычно это статические helper или `@Component` утилитарного характера.

Если в сервисе выделяется больше одной доменной области, например, `User` и `Wallet`, то все классы (`web`, `mappings`,
`models`, `entities`, `repositories`, `services`), связанные с ними, должны находится в отдельном пакете user и wallet
соответственно.

Для того чтобы сервис (`@Service`) был изолированный, он должен взаимодействовать только с DAO и репозиториями из своего
домена. Т.е. если нам в `WalletService` нужно получить пользователя, то мы должны использовать `UserService`, а не
работать напрямую с `UserRepository`. Иначе нарушается Single Responsibility принцип и сильно усложняются unit тесты.

Если есть какие-то общие классы, сервисы, то они выносятся в пакет `common`.

#### Разбиение по доменным сущностям

Делать подпакеты можно, например request / response в `models`, но если требуются подпакеты в `service`, обычно это
значит что требуется разбиение на более мелкие доменные области.

Доменная область обычно 1 к 1 связана с бизнес-процессом, т.е. у вас в одном пакете есть контроллеры (и сервисы),
которые выполняют разную функциональность из разных Use Case (работа с пользователем (создание, блокировка) и работа с
кошельком (создание, пополнение, закрытие)), то это обычно различные доменные области.

Если у вас есть необходимость в sql / jpa запросе использовать join на таблицы из разных доменных областей, то лучше это
делать в java коде, потому что в случае возможного распила сервиса на части, части этого запроса могут относиться к
разным доменным областям, а значит join придется распиливать.

К одной доменной сущности могут относиться объекты, которые будут невалидны без основной сущности. Например, `User` ->
`Address`, адрес будет невалиден без привязки к пользователю, но `Address` -> `Country`, `Address` -> `City` уже не
будут в одном домене, т.к. `Country` и `City` могут потребоваться в других процессах.

#### Семантическое версионирование

[Версионированиеи общих модулей](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=1274953383)

#### Описание `@Entity`

* При описании `@Entity` данных всегда описываем `@Table` с именем таблицы в базе данных.
* Все описания колонок в `@Entity` должны максимально соответствовать колонкам в базе данных. При описании колонки в
  аннотации `@Column` всегда указывать `name` и дополнительные аргументы, например `nullable`, `length` (для `VARCHAR`),
  `scale` и `presicion` (для `NUMBER`). Если для колонки задается `DEFAULT`, то он описывается
  в `columnDefinition = "BOOLEAN NOT NULL DEFAULT false""`.
* В entity всегда руками описывать `equals`, `hashCode`, `toString`. Использовать `@Data` в `@Entity` запрещено.
  Аналогично моделям используем `EqualsBuilder`, `HashCodeBuilder`, `ToStringBuilder` Apache Commons Lang 3.
* В `@Entity` можно использовать аннотации `@Getter`, `@Setter`, сборка сущности через `@Builder`.
* Если `@Entity` ссылается на другую сущность, то в `@JoinColumn` всегда описываем имя колонки (`name`), которая
  ссылается на другую таблицу и `@ForeignKey` с именем constraint, совпадающим с DDL.
  ```jshelllanguage
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "calculation_id", foreignKey = @ForeignKey(name = "fk_calculation_version_calculation_id"))
  private CalculationEntity calculation;
  ```

#### Использование прямых ссылок на другую таблицу

Если есть необходимость сделать запрос к другой сущности через id, например получить все `CalculationVersionEntity`, у
которых `CalculationEntity :: id == 1`, то в HQL этот запрос будет выглядеть вот так:

```java

@Getter
@Setter
@Accessors(chain = true)
@Entity
@Table(name = "calculation_version")
@EntityListeners(AuditingEntityListener.class)
public class CalculationVersionEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

  ...

    @Column(name = "calculation_id", updatable = false, insertable = false)
    private Long calculationId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "calculation_id", foreignKey = @ForeignKey(name = "fk_calculation_version_calculation_id"))
    private CalculationEntity calculation;

  ...
```

```hql
select cv from CalculationVersionEntity cv where cv.calculation.id = :calculationId
```

Это может привести к созданию лишнего `JOIN`:

```postgresql
SELECT cv.*
FROM calculation_version cv
         INNER JOIN calculation c ON cv.calculation_id = c.id
WHERE c.id = :calculationId
```

В дополнение к `@JoinColumn` можно описать
поле `@Column(name = "calculation_id", updatable = false, insertable = false)`, которое будет маппироваться реальное
поле в `calculation_id` в базе данных, и HQL запрос выше можно описать так:

```hql
select cv from CalculationVersionEntity cv where cv.calculationId = :calculationId
```

Поле `calculationId` заполнять руками нельзя (оно `updatable = false`, `insertable = false`), оно
заполняется `Hibernate` после создания записи.

#### Правила описания `@OneToMany`, `@ManyToOne`, `@ManyToMany`

По-умолчанию для `@OneToMany`, `@ManyToOne`, `@ManyToMany` используем тип загрузки `FetchType.LAZY`.
Оставлять `FetchType.EAGER` в `@ManyToOne` можно только можно в случае, если основная сущность всегда нужна в связке с
дочерней.

Для связи `@OneToMany` `LAZY` по дефолту, его прописывать не нужно.

Для связи `@OneToOne` реализовать `LAZY` без костылей нельзя. Но обычно использование `@OneToOne` означает, что дочерняя
сущность является частью общей сущности, что не противоречит описанному выше правилу.

Изменение типа связи с `LAZY` на `EAGER` крайне не рекомендуется, это может очень негативно сказаться на
производительности, т.к. при поднятии одной сущности, будут подниматься еще N дополнительных сущностей.

При этом, если связь помечена `LAZY`, а обращение к ней выполняется вне транзакции, то будет выброшен
`LazyInitializationException`. Для предотвращения такой ситуации нужно явно использовать транзакции и (или)
использовать `join fetch` и `@EntityGraph` в случае, когда эти данные нужны в получаемом результате.

Аннотация `@JoinColumn` может быть по обе стороны связи (`@OneToMany`, `@ManyToOne`), она определяет главную сущность в
отношении.

Подробнее в
статье [Использование библиотеки Hibernate](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=1140483682).

Пример использования [jpa-example](https://github.com/Romanow/jpa-example/).

#### JPA Auditing

Если требуется использовать поля: дата создания, дата последней модификации, имя пользователя, который создал или
изменил запись, то нужно использовать стандартные средства JPA Auditing с помощью аннотаций:

* `@CreatedDate`, `@LastModifiedDate`
* `@CreatedBy`, `@LastModifiedBy`

Для их использования нужно создать `DateTimeProvider` и в самом `@Entity` в `@EntityListeners` прописать
использование `AuditingEntityListener.class`.

```java

@Configuration
@EnableJpaAuditing(dateTimeProviderRef = "dateTimeProvider")
public class DatabaseConfiguration {

    @Bean
    public DateTimeProvider dateTimeProvider() {
        return CurrentDateTimeProvider.INSTANCE;
    }
}
```

```java

@Getter
@Setter
@Accessors(chain = true)
@Entity
@Table(name = "assessment_document")
@EntityListeners(AuditingEntityListener.class)
public class AssessmentDocumentEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private LocalDateTime createdDate;

    @LastModifiedDate
    @Column(name = "modified_date", nullable = false)
    private LocalDateTime modifiedDate;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        AssessmentDocumentEntity that = (AssessmentDocumentEntity) o;
        return Objects.equal(name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(name);
    }

    @Override
    public String toString() {
        return MoreObjects.toStringHelper(this)
                .add("id", id)
                .add("name", name)
                .add("createdDate", createdDate)
                .add("modifiedDate", modifiedDate)
                .toString();
    }
}
```

#### Форматирование SQL

* Зарезервированные слова _SQL_ пишутся с большой буквы.
* Зарезервированные слова в _HQL_ пишутся с маленькой буквы.
* Имена баз данных, схем, таблиц и полей пишутся в snake_case.
* Имя базы `<префикс_calm>_<название_сервиса>_<опциональный_суффикс>`. Для основной базы, то суффикс не нужен (тем более
  если она одна). Например, для сервиса хранения результатов `calm-cash-flow-result-holder` имя базы будет:
  `calm_cash_flow_result_holder`.
* Имя схемы `public`, `staged`. Если в одной базе несколько разнородных схем (как сейчас `sudo`, `data_preparation`,
  `incoming_document`), то `<название_сервиса>_<опциональный_суффикс>`. Суффикс нужен, если схема не основная, например:
  база данных `calm_subo`, схема `cash_flow_result_holder` (основная) и `cash_flow_result_holder_staged` (куда DRP
  выгружает данные).
* При создании таблицы ограничения `CHECK`, `CONSTRAINT` по возможности описываются вместе с полем, к которому
  относятся.
* Все названия специальных объектов должны иметь имена: `<prefix>_<table-name>_<field-name>`.

```postgresql
CREATE SEQUENCE seq_calculation_priority AS BIGINT START WITH 1 INCREMENT BY 10;

CREATE TABLE calculation
(
    id                  BIGSERIAL PRIMARY KEY,
    uid                 UUID         NOT NULL,
    name                VARCHAR(255) NOT NULL,
    description         VARCHAR(255),
    priority            INT          NOT NULL DEFAULT NEXTVAL('seq_calculation_priority'),
    forecasting_horizon INT          NOT NULL CHECK (forecasting_horizon > 0)
);

CREATE INDEX idx_calculation_name ON calculation (name);
CREATE UNIQUE INDEX ui_calculation_uid ON calculation (uid);

CREATE TABLE calculation_status_history
(
    id             BIGSERIAL PRIMARY KEY,
    status         VARCHAR(80),
    calculation_id INT
        CONSTRAINT fk_calculation_status_history_calculation_id
            REFERENCES calculation (id) ON DELETE NO ACTION
);

CREATE INDEX idx_calculation_status_history_calculation_id ON calculation_status_history (calculation_id);
```

Префиксы объектов:

* `index` – `idx_`;
* `unique index` – `udx_`;
* `sequence` – `seq_`;
* `foreign key` – `fk_`.

Для всех Foreign Key создаем индексы.

#### Использование DEFAULT в DDL

Использование DEFAULT на уровне DDL крайне нежелательно, т.к. это:

* снижает прозрачность данных, т.к. данные в запросе `INSERT` отличаются от того, что попадает в таблицу;
* увеличивается объём хранимых данных;
* при частой вставке сильно бьет по производительности, т.к. на каждую строку вызывается обработчик.

Значения по-дефолту задаем на уровне кода.

#### Использование Primary и Foreign Key для обеспечения связности данных внутри реляционной базы данных

В реляционных базах данных Primary Key и Foreign Key используются для контроля целостности данных. Это позволяет
блокировать удаление строки, если на нее ссылаются другие записи.

Менять дефолтное поведение `ON UPDATE` / `ON DELETE` на `CASCADE` не надо, т.к. это неявно грохнуть половину базы
данных.

Для Primary Key индекс [создается по-умолчанию](https://www.postgresql.org/docs/current/indexes-unique.html). Для
Foreign Key обязательно создавать индекс, иначе `JOIN` между таблицами будет выполняться как `Sequence Scan`.

#### Использование UUID для взаимодействия между сервисами

Если требуется создать связь сущностей между сервисами, то _дополнительно_ к `private key` создается
поле `uid UUID NOT NULL`, которое является внешним ключом между базами данных.

Т.е. если требуется вывести какую-то информацию о сущности на UI или сделать ссылку на нее из другого сервиса, то в
сущности заводится поле uid, и все взаимодействие идет через него. Внутренние `primary key` используются только для
ссылок _внутри одной_ базы данных.

Т.к. uid должен быть уникальным ключом, на него создается `unique index`:

```postgresql
CREATE TABLE calculation
(
    id  BIGSERIAL PRIMARY KEY,
    uid UUID NOT NULL
);

CREATE UNIQUE INDEX udx_calculation_uid ON calculation (uid); 
```

#### Использование профилей

Разделение на профили нужно для прозрачности настройки приложения в разных средах. Разбиение по профилям выполняется
только на уровне профиля приложения, используются стандартные механизмы Spring.

Для каждого контура (`dev`, `ift`, `psi`, `prod`) описывается свой собственный профиль. Credential к базе данных,
сертификаты `ArtemisMQ` и остальные важные настройки будут задаваться через переменные среды, которые будут браться из
Secret и ConfigMap.

На промежуточных средах (`dev`, `ift`) база данных одна, она разворачивается на отдельной машине, но для каждого сервиса
создается своя схема. Соответственно в настройках приложения надо указывать схему на уровне конфигурации, а не в коде.

Подробнее в статье [Настройка Зависимостей](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=806190975).

#### Использование Shared модулей

Если в проекте есть API, то в нем реализуются два модуля:

* `shared` – модуль содержит _только_ модели, используемые в запросах и ответах API. Внутренние модели описываются
  в `assembly`.
* `assembly` – основной код сервиса, содержит в зависимостях модуль `shared` как `implementation project(":shared")`.

`shared` модуль не должен содержать лишних зависимостей, в нем категорически запрещено ссылаться на другие `shared`
модули. Если в API требуются модели из другого модуля, то либо модели дублируются в этом `shared` модуле, либо клиент
должен будет подключать оба `shared` модуля.

#### Использование валидации методов JSR303

Любые входные параметры должны проходить валидацию в контроллере с помощью аннотаций `javax.validation.constraints`.
Валидация может быть двух видов:

##### Валидация аргументов метода в контроллере

Если требуется проверить аргументы метода в контроллере, то на контроллер ставится аннотация `@Validated` и на аргументы
ставятся необходимые аннотации `@NotEmpty`, `@Positive`, `@Pattern` и т.д. Аннотации `@NotNull` в аргументах
бессмысленны, т.к. `@RequestParam`, `@QueryParam` имеет параметр `required = true` по-умолчанию, т.е. при вызове метода
эти аргументы уже точно не `null`.

Если аргумент должен иметь определенный тип (например `Double`, `UUID` и т.д.), то нужно сразу ставить целевой этот тип
данных, а не принимать все в `String`, а потом паттернами проверять соответствие типу. Т.к. наше API для внутреннего
использования, мы можем сразу указывать необходимый тип данных, и в случае несоответствия аргумента типу просто
возвращать 500 ошибку.

```java

@Validated
@RestController
@RequestMapping("/api/v1/calculation")
public class CalculationController {
    private final CalculationService calculationService;

    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public List<CalculationResponse> search(@RequestParam @NotEmpty String name,
                                            @RequestParam UUID factDataUid) {
        return calculationService.search(name, factDataUid);
    }
  
  ...
}
```

##### Валидация полей в объекте

Для валидации полей в объекте, в контроллере валидируемый объект помечается аннотацией `@Valid`, а в самом объекте
аннотации ставятся над полями.

```java

@Data
@Accessors(chain = true)
public class CalculationCreateRequest {
    @NotEmpty
    private String name;

    private String description;

    @NotNull
    private UUID factDataUid;

    @NotNull
    @Positive
    private Integer balanceSheetForecastingHorizon;

    @NotNull
    private String type;
}
```

```java

@RestController
@RequestMapping("/api/v1/calculation")
public class CalculationController {
    private final CalculationService calculationService;

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEnity<Void> create(@Valid @RequestBody CalculationCreateRequest request) {
        final UUID uid = calculationService.create(request);
        URI uri = ServletUriComponentsBuilder.fromCurrentContextPath()
                .path(API_1 + CALCULATION + "/{uid}")
                .buildAndExpand(calculation.getUid())
                .toUri();
        return ResponseEntity.created(uri).body(calculation);
    }
}
```

##### Обработка ошибок валидации

Для обработки ошибок валидации в `@RestControllerAdvice`
описываем метод, аннотированный `@ExceptionHandler(MethodArgumentNotValidException.class)`, который получает
исключение `MethodArgumentNotValidException`, которое в себе хранит `BindingResult` с информацией о всех ошибках
валдиации.

```java

@Slf4j
@RestControllerAdvice
public class ExceptionController {

    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ValidationErrorResponse notFound(MethodArgumentNotValidException exception) {
        final BindingResult bindingResult = exception.getBindingResult();
        return new ValidationErrorResponse(buildMessage(bindingResult), buildErrors(bindingResult));
    }

    private String buildMessage(BindingResult bindingResult) {
        return String.format("Error on %s, rejected errors [%s]",
                bindingResult.getTarget(),
                bindingResult.getAllErrors()
                        .stream()
                        .map(DefaultMessageSourceResolvable::getDefaultMessage)
                        .collect(joining(",")));
    }

    private List<ErrorDescription> buildErrors(BindingResult bindingResult) {
        return bindingResult.getFieldErrors()
                .stream()
                .map(e -> new ErrorDescription(e.getField(), e.getDefaultMessage()))
                .collect(toList());
    }
}
```

##### Переводы сообщений валидации

Аннотации валидации JSR-303 имеют параметр `message`, который ссылается на `ValidationMessages.properties` внутри
модуля `org.hibernate.validator:hibernate-validator`.

```java

@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
public @interface NotEmpty {
    String message() default "{javax.validation.constraints.NotEmpty.message}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

Для описания своих сообщений валидации, создаем файл `resources/ValidationMessages_<locale>.properties` и в нем
описываем коды и переводы по аналогии с `messages.properties`.

При использовании аннотаций указываем параметр `@NotEmpty(message = "{field.not.empty}")` со ссылкой на коды. Важно
заметить, что код должен быть указан в фигурных скобках, без этого `message` будет воспринят как готовый перевод.

В сообщениях валидации можно использовать аргументы из аннотации валидации.

```properties
# ValidationMessages_en.properties
field.not.empty=Field must be not empty
field.fixed.size=Field must be between interval {min} to {max}
```

```java

@Data
@Accessors(chain = true)
public class CalculationCreateRequest {
    @NotEmpty(message = "{field.not.empty}")
    private String name;

    @NotNull
    @Size(min = 0, max = 100, message = "{field.fixed.size}")
    private Integer balanceSheetForecastingHorizon;
}
```

#### Маппинг HTTP кодов ответов на ошибки сервиса

TODO

#### Обработка exceptions

Если нужно сделать переводы сообщений, то можно воспользоваться стандартным механизмом Spring i10n.

* Создается файл `resources/messages_<locale>.properties`, где locale – сокращенное (`ru`, `en`) или полное имя (`ru_RU`
  , `en_US`) локали.
  ```properties
  # messages_ru.properties
  product.code.not.found=Код продукта не найден
  ```
  ```properties
  # messages.properties
  product.code.not.found=Product code not found
  ```
* В `resources/application.properties` прописывается `spring.messages.basename=messages` (по-умолчанию).
* Создаем сервис `MessageHelper`, если `Locale` не передается привызове метода, то
  используется `LocaleContextHolder.getLocale()`, который берет Locale из параметров запроса.
  ```java
  @Component
  @RequiredArgsConstructor
  public class MessageHelperImpl
        extends MessageHelper {
    private final MessageSource messageSource;
  
      @NotNull
      public String tr(@NotNull String code) {
          return tr(code, null, LocaleContextHolder.getLocale());    
      }
  
      @NotNull
      public String tr(@NotNull String code, @NotNull Object[] args) {
          return messageSource.getMessage(code, args, LocaleContextHolder.getLocale());
      }
  
      @NotNull
      public String tr(@NotNull String code, @NotNull Object[] args, @NotNull Locale locale) {
          return messageSource.getMessage(code, args, locale);
      }
  }
  ```
* В `ExceptionType` вместо перевода содержится `code`:
  ```jshelllanguage
  public enum ExceptionType {
    UNSUPPORTED_ERROR("unsupported.operation"),
    PRODUCT_CODES_NOT_FOUND("product.code.not.found"),
    ...
  }
  ```
* В месте обработки `@RestControllerAdvice` в лог пишется в `Locale.ENGLISH`, а пользователю отдается в соответствии с
  его локалью браузера.
  ```java
  @Slf4j
  @RestControllerAdvice
  @RequiredArgsConstructor
  public class ExceptionController {
      private final MessageHelper messageHelper;
  
      @ExceptionHandler(ProductCodeNotFoundException.class)
      public ResponseEntity<ErrorResponse> notFound(ProductCodeNotFoundException exception) {
          log.error(messageHelper.tr(exception.code(), exception.args(), Locale.ENGLISH));
          return ResponseEntity
                  .status(HttpStatus.NOT_FOUND)
                  .body(new ErrorResponse(messageHelper.tr(exception.code(), exception.args()), exception.getCode(), applicationName));
      }
  }
  ```

#### Миграции в `liquidbase`

Для миграций используем [`liquidbase`](https://docs.liquibase.com/concepts/introduction-to-liquibase.html).

Именование файлов миграции `YYYYMMDD_HHMI_<описание_изменений>.xml`: `20211231_2359_drop_database.xml`

Имена шагов миграций внутри `liquidbase` состоит из `<task-number>-<NN>-<описание-изменений>`.

Т.к. на psi и prod у нас нет доступа до БД, то миграции должны выполняться аккуратно и ничего не сломать. Для увеличения
надежности используем:

* `preCondition` – проверка того, что скрипт нужно выполнять. Это очень полезный инструмент, т.к. он предотвращает
  повторное создание уже существующих объектов.
* `rollback` – скрипт отката, нужен по двум причинам:
    * если изменения были применены по-ошибке или содержат ошибку;
    * при работе тестировщиков на тестовом контуре при взятии в работу задачи с другим набором миграций.

Комментарии к простым миграциям не нужны. Если скрипт сложный или объемный (более 1 экрана), то комментарий обязателен.

Используем описание миграций в xml, если выполняется изменение DDL. Если требуется модификация данных, то используем
SQL.

```xml

<changeSet id="ALMCALC-777-01-update-currency-code-for-rubles" author="<автор>">
    <preConditions onFail="MARK_RUN" expectedResult="1">
        SELECT COUNT(1) FROM currency WHERE id = 'RUB' and name='Российский рубль'
    </preConditions>

    <comment>Понятное описание того, что делает данный sql</comment>

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

#### Использование MapStruct

Если в проекте требуется делать преобразования `entity` -> `response model`, `request model` -> `entity`, то нужно
использовать [`MapStruct`](https://mapstruct.org/documentation/stable/reference/html/).

Задача мапперов просто переложить готовые данные из одного объекта в другой. Мапперы следует делать быть максимально
простыми, в идеальном случае они должны заниматься только перекладыванием плоских полей (`String`, `Integer`,
`BigDecimal`) из объекта в объект.

Если объект является составным, то для каждой доменной сущности должен быть написан свой mapper. Структрурная
зависимость мапперов должна повторять структуру доменной области для простоты понимания, поддержки и соблюдения Single
Responsibility.

Для сложных составных объектов нужно разделять операции создания и редактирования.

* Для связи мапперов использовать `uses`: `@Mapper(uses = { AddressMapper.class }`.
* Если ваше приложение написано на Spring Boot, то мапперы тоже должны быть под управлением
  Spring: `@Mapper(componentModel = "spring")`.
* Для создания мапперов
  использовать `@Mapper(componentModel = "spring", injectionStrategy = InjectionStrategy.CONSTRUCTOR)`, что позволит
  создавать маппер в тестах без создания контекста Spring.
* Нужно избегать внедрения других сервисов в мапперы, это упростит поддержку. Лучше сделать что маппер сможет, а
  остальное уже доставить в сервисе вызова и туда внедрить зависимости.
* Не рекомендуется использовать `@BeforeMapping`, так как входная (source) сущность может находиться
  в `Persistence Context` и её изменения в рамках метода `@BeforeMapping` могут быть неявно сохранены.
* Не использовать `expression="java()"` для методов с бизнес-логикой – это очень сложно тестировать.
* Полезно использовать `@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR)`, а все неиспользуемые поля в mapping явно
  указывать через `@Mapper(ignore = true, target = "..."")`. Это позволит не пропустить поля в итоговом объекте.

Подробнее в
статье [Использование библиотеки Hibernate](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=1140483682).

#### В OpenAPI поддерживать консистентность по ошибкам в `@ApiResponse`

Используем OpenAPI версии 3, реализация [`springdoc`](https://springdoc.org/).

При добавлении новых ошибок при обработке вызова, они должны быть отображены в `@ApiResponse`. Это нужно, чтобы
декларация метода в OpenAPI была консистентна с кодом, т.е., чтобы при вызове метода не возникало не описанных кодов
ответов (кроме 500).

#### Использование RestTemplate vs. WebClient

TODO

#### Вынесение транспортных запросов в отдельный сервис

Работу с `WebClient` (`RestTemlate`) убираем в отдельны сервис, который инкапсулирует в себе всю логику сборки запроса,
вызова и обработки результатов запроса. Сервис имеет суффикс `Client`.

```kotlin
@Service
class WarehouseServiceImpl(
        private val warehouseWebClient: WebClient
) : WarehouseService {

    override fun takeItem(orderUid: UUID, model: String, size: SizeChart): Optional<OrderItemResponse> {
        val request = OrderItemRequest(orderUid, model, size.name)
        return warehouseWebClient
                .post()
                .body(BodyInserters.fromValue(request))
                .retrieve()
                .onStatus({ it == NOT_FOUND }, { response -> buildEx(response) { EntityNotFoundException(it) } })
                .onStatus({ it == CONFLICT }, { response -> buildEx(response) { ItemNotAvailableException(it) } })
                .onStatus({ it.isError }, { response -> buildEx(response) { WarehouseProcessException(it) } })
                .bodyToMono(OrderItemResponse::class.java)
                .blockOptional()
    }
}
```

#### Properties vs. YAML

TODO: Обсудить с командой. Мне кажется YAML нагляднее

#### Префиксы путей в константах

TODO: Обсудить с командой

#### Использование @ConfigurationProperties vs. @Value

TODO

#### Использование готовых библиотек, а не разработка своих решений

Для решения типовых задач ищем и используем готовые библиотеки с высоким рейтингом. Например, для проверки наличия
непустых символов в строке можно написать свой метод `hasChars`:

```jshelllanguage
public boolean hasChars(@Nullable String str) {
    return str != null && str.chars().anyMatch(c -> c != 0);
}
```

Но на этот метод нужно написать тесты и проверить, что реализация оптимальна. Но лучше использовать готовый
метод `org.springframework.util.StringUtils.hasText(@Nullable String str)`. Т.к. библиотеку Spring поддерживает
community, реализация таких методов будет не хуже, чем самописная.

Аналогичная ситуация с самописными реализациями:

* транслитерация rus -> latin: [Iuliia](https://github.com/Homyakin/iuliia-java);
* сборка SQL для записи скрипта в файл: [SqlBuilder](https://openhms.sourceforge.io/sqlbuilder/).

#### Настройка проекта

* gradle: описание используемых плагинов.
* Настройка интеграции с системными сервисами.
* Выключение интеграции для локального профиля.

[Шаблон сервисов](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=801089821)
[Настройка зависимостей](https://wiki.corp.dev.vtb/pages/viewpage.action?pageId=806190975)