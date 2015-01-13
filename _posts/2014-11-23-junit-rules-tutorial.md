---
layout: post
title:  "Механизм правил в JUnit"
date:   2014-11-22 19:00:22 +0400
author:
    name: Kalinin Yuri
    email: kurau.mm@yandex.ru
categories: [junit]
tags: [junit, rule]
comments: true
published: true
---

## JUnit

С фрэймворком модульного тестирования [JUnit](http://junit.org/), кажется, знакомы все. 
Как видно из его описания, он направлен на создание "повторяемых" (repeatable) тестов. Что это значит?
Значит тесты, которые мы написали, придётся ещё и поддерживать.
Поэтому в первую очередь стоит рассматривать данный фреймворк как набор удобных механизмов, 
которые направлены на то, чтобы облегчить написание и дальнейшую поддержку наших тестов.
В данной статье мы остановимся подробней на механизме правил - Rules (далее просто Рул).
Артём Кошелев ранее в своём [блоге](http://artkoshelev.github.io) уже писал о них ([Рулы Рулят](http://artkoshelev.github.io/posts/rules-rules/)), но здесь мы попытаемся копнуть глубже.

## Fixture

Test fixture — это особое состояние данных необходимое для успешного выполнения теста. 
Допустим, есть два или более теста, которые работают с одинаковым набором данных (fixture).
Чтобы подготовить эти данные для каждого отдельного теста в классе необходимо воспользоваться 
специальными аннотациями `@Before` и `@After`. Методы с такими аннотациями выполняются, соответственно, 
"до" и "после" каждого теста:

{% highlight java %}

public class SimpleTest {

    @Before
    public void before() {
        System.out.println("before");
    }
     
    @After
    public void after() {
        System.out.println("after");
    }
     
    @Test
    public void test() {
        System.out.println("test");
    }
        
    @Test
    public void test2() {
        System.out.println("test2");
    }
}

{% endhighlight %}

Результат выполнения будет следующим:
> before  

> test  

> after  

> before  

> test2  

> after

`@Before` и `@After` являются самыми простыми рулами, вшитыми в ядро JUnit. Наравне с ними существуют рулы `@BeforeClass` и `@AfterClass`, которые работают аналогичным образом и вызываются для целого класса, а не для каждого метода:

{% highlight java %}

public class SimpleTest {

    @BeforeClass
    public static void before() {
        System.out.println("before");
    }
     
    @AfterClass
    public static void after() {
        System.out.println("after");
    }
     
    @Test
    public void test() {
        System.out.println("test");
    }
        
    @Test
    public void test2() {
        System.out.println("test2");
    }
}

{% endhighlight %}

Получим следующий результат:
> before  

> test  

> test2  

> after

Заметим, что методы с `@BeforeClass` и `@AfterClass` должны быть статическими. 

## Rules

### Custom Rules
Допустим теперь мы хотим использовать предыдущий код (методы before и after) в других классах. 
Вместо того, чтобы копировать методы целиком или выносить в отдельный базовый класс для каждого нового
набора тестов, создадим собственную рулу.

Рула представляет из себя класс, реализующий интерфейс `org.junit.rules.TestRule`. Для того, чтобы создать новую рулу необходимо реализовать метод apply, возвращающий объект типа `org.junit.runners.model.Statement`.

Далее, каждый индивидуальный вызов теста в JUnit представляет собой вызов метода `evaluate()` как раз этого объекта типа `Statement`. В руле мы просто оборачиваем выполнение теста (`base.evaluate()`) своим кодом:

{% highlight java %}

public class SimpleRule implements TestRule {

    @Override
    public Statement apply(final Statement base, Description description) {
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                System.out.println("before");
                base.evaluate();
                System.out.println("after");
            }
        };
    }
}

{% endhighlight %}

Для использования рулы у себя необходимо подключить её как поле соответствующего класса с аннотацией `org.junit.Rule` или статическое поле с аннотацией `org.junit.ClassRule`, в зависимости от поставленной цели. При запуске теста JUnit будет ориентироваться только по этим аннотациям и рулы без них (если такие имеются) будут проигнорированы.  

{% highlight java %}

public class SimpleTest {

    @Rule
    public SimpleRule rule = new SimpleRule();
     
    @Test
    public void test() {
        System.out.println("test");
    }
        
    @Test
    public void test2() {
        System.out.println("test2");
    }
}

{% endhighlight %}

Результат:
> before  

> test  

> after  

> before  

> test2  

> after


### Base Rules

* [External Resources](#external_r)
  * [Temporary Folder](#tfolder)
* [Test Watcher](#watcher)
  * [Test Name](#tname)
* [Verifier](#verifier)
  * [Error Collector](#ecollector)
* [Expected Exception](#expectedexcptn)
* [Timeout](#timeout)
* [Rule Chain](#rulechain)

Прежде чем писать свои собственные рулы следует познакомиться с уже существующими. Фреймворк предлагает несколько готовых рул с удобными методами, которые можно использовать "из коробки".
Самой популярной из них, как мне кажется, является `org.junit.rules.ExternalResource`:

<a name="external_r"/>
#### ExternalResource

Данную рулу предполагается использовать в тех случаях, когда подготовленные для тестирования данные нобходимо очистить (освободить) при любом исходе теста. Посмотрим как это реализовано:

{% highlight java %}
@Override
public void evaluate() throws Throwable {
    before();
    try {
        base.evaluate();
    } finally {
        after();
    }
}
{% endhighlight %}

Здесь выделим только метод evaluate, чтобы показать сходство с предыдущим примером. Методы
`before()` и `after()` предполагается реализовать самому. В них и нужно описать управление своими данными. 

<a name="tfolder"/>
##### TemporaryFolder

Рула `org.junit.rules.TemporaryFolder` является частным случаем ExternalResource, и позволяет создавать файлы и папки, которые гарантированно удалятся после завершения теста. Пример использования с сайта производителя:

{% highlight java %}

@Rule
public TemporaryFolder folder = new TemporaryFolder();

@Test
public void testUsingTempFolder() throws IOException {
    File createdFile = folder.newFile("myfile.txt");
    File createdFolder = folder.newFolder("subfolder");
    // ...
}

{% endhighlight %}

<a name="watcher"/>
#### TestWatcher

Следущая базовая рула `org.junit.rules.TestWatcher` не менее популярна и призвана добавить немного свободы в наши тесты, так как она (здесь я не буду привдить её реализацию) предоставляет возможность переопределить следующие методы:

> succeeded

> failed

> skipped

> starting

> finished

По названиям методов можно догадаться, в какой момент они выполнятся, а именно: начало или конец теста, успешное или не успешное завершение.

TestWatcher отлично подходит для сбора информации о тесте, так как в каждый метод подаётся `org.junit.runner.Description`. При этом стоит быть осторожным с добавлением логики в эти методы, так как любое исключение будет обработано и напечатано только в конце теста:

{% highlight java %}

@Rule
public TestWatcher watcher = new TestWatcher() {

    @Override
    protected void starting(Description description) {
        System.out.println("starting");
        throw new IllegalStateException();
    }

    @Override
    protected void finished(Description description) {
        System.out.println("finished");
    }
};

@Test
public void test() {
    System.out.println("test");
}

{% endhighlight %}

В результате получим такой вывод
> starting

> test

> finished

плюс сообщение об ошибке.

<a name="tname"/>
##### TestName

Простая для понимания рула `org.junit.rules.TestName` является наследником TestWatcher и позволяет использовать имя метода внутри него самого: 

{% highlight java %}
private String name;

@Override
protected void starting(Description d) {
    name = d.getMethodName();
}

public String getMethodName() {
    return name;
}

{% endhighlight %}

<a name="verifaer"/>
#### Verifier

Класс `org.junit.rules.Verifier` также как и ExternalResource является базовым классом, в котором предполагается реализовать один метод `verify()`:

{% highlight java %}

@Override
public void evaluate() throws Throwable {
    base.evaluate();
    verify();
}

{% endhighlight %}

<a name="ecollector"/>
##### ErrorCollector

Как пример использования Verifier рассмотрим рулу `org.junit.rules.ErrorCollector`, которая разрешает 
"продолжить выполнение теста после первой ошибки". Использование этой рулы позволит, например, собрать все ошибки произошедшие в тесте в одном отчёте. Хотя при правильном формировнии тесткейсов (один тест - одна проверка) необходимости в такой руле нет. 

{% highlight java %}

@Rule
public ErrorCollector collector= new ErrorCollector();

@Test
public void example() {
    collector.checkThat("Должны совпадать", "1", is("3"));
    collector.checkThat("Должны совпадать", "1", is("1"));
    collector.checkThat("Должны совпадать", "1", is("2"));
}

{% endhighlight %}

Метод `checkThat( ... )` является обёрткой для стандартной проверки `assertThat( ... )`, но в отличие от последнего не прерывает выполнение теста если проверка не прошла. Результатом такого кода будет отчёт, который содержит в себе ошибки первой и третьей проверки.

<a name="expectedexcptn"/>
#### ExpectedException

Проверка кода на предмет правильной работы в исключительных ситуациях является одной из важных задач в тестировании. В JUnit есть возможность проверить, что в процессе выполнения бросается нужное исключение:

{% highlight java %}

@Test(expected=NullPointerException.class)
public void throwsNullPointerExceptionWithMessage() { 
    System.out.println("test");
}

{% endhighlight %}

Рула `org.junit.rules.ExpectedException` расширяет этот функционал и позволяет проверить не только класс бросаемого исключения:

{% highlight java %}

@Rule
public ExpectedException thrown = ExpectedException.none();

@Test
public void throwsNullPointerExceptionWithMessage() {
    thrown.expect(NullPointerException.class);
    thrown.expectMessage("happened?");
    thrown.expectMessage(startsWith("What"));
    throw new NullPointerException("What happened?");
}

{% endhighlight %}

<a name="timeout"/>
#### Timeout

Иногда встречаются тесты в которых кроме проверок основной функциональности требуется следить за продолжительностью их выполнения, и если тот или иной сценарий выполняется дольше заданного времени (ответ сервера, отрисовка веб страницы) нужно выдавать ошибку. В единичных случаях можно добавить timeout в аннотацию @Test:

{% highlight java %}

@Test(timeout = 1000)
public void test() { 
    System.out.println("test");
}

{% endhighlight %}

Если нужно распространить заданный `timeout` на все тесты в классе воспользуемся рулой `org.junit.rules.Timeout`:

{% highlight java %}

@Rule
public TestRule timeout = new Timeout(1000);

@Test
public void test() throws Exception { 
    System.out.println("test");
    Thread.sleep(500); 
}

@Test
public void test2() throws Exception {
    System.out.println("test2");
    Thread.sleep(1500); 
}

{% endhighlight %}

Упадёт только второй тест с соответствующей ошибкой "test timed out after 1000 milliseconds".


<a name="rulechain"/>
#### RuleChain

Если в тесте есть несколько рул, то вероятнее всего (на самом деле нет) они будут выполняться в том порядке, в котором встречаются в коде. В действительности, порядок, в котором они будут вызваны, зависит от реализации JVM.

Рассмотрим обычную ситуацию, когда есть рула для логирования обращений к некоторому серверу и рула для установки соединения с ним. Очевидно, что сначала должна стартовать рула для сбора логов, чтобы информация о коннекте к серверу была записана. 

В случае, когда нужно использовать несколько рул в строго определённом порядке на помощь приходит `org.junit.rules.RuleChain`:

{% highlight java %}

@Rule
public TestRule chain= RuleChain
                       .outerRule(new SimplePrintRule("log rule"))
                       .around(new SimplePrintRule("connect rule"))
                       .around(new SimplePrintRule("some other rule"));

@Test
public void test() {
    System.out.println("test");
}

public class SimplePrintRule extends TestWatcher {
    private String name;

    public SimplePrintRule(String name) {
        this.name = name;
    }

    @Override
    protected void starting(Description description) {
        System.out.println("starting " + name);
    }

    @Override
    protected void finished(Description description) {
        System.out.println("finished " + name);
    }
}

{% endhighlight %}

В результате получим порядок:
> starting log rule

> starting connect rule

> starting some other rule

> test

> finished some other rule

> finished connect rule

> finished log rule




Подробней о рулах также можно узнать из репозитория проекта на гитхабе https://github.com/junit-team/junit

