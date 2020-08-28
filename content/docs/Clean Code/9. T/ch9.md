# 第 9 章 Unit Tests 单元测试

![](figures/ch9/9_1fig_martin.jpg)

Our profession has come a long way in the last ten years. In 1997 no one had heard of Test Driven Development. For the vast majority of us, unit tests were short bits of throw-away code that we wrote to make sure our programs “worked.” We would painstakingly write our classes and methods, and then we would concoct some ad hoc code to test them. Typically this would involve some kind of simple driver program that would allow us to manually interact with the program we had written.

> 过去十年以来，编程专业领域进步很大。1997 年时，没人听说过测试驱动开发。对于我们之中的大多数人来说，单元测试是那种用来确保程序“可运行”的用过即扔的短代码。我们辛勤地编写类和方法，再弄出一些特殊代码来测试它们。通常这会是种简单的驱动式程序，让我们能够手工与自己编写的程序交互。

I remember writing a C++ program for an embedded real-time system back in the mid-90s. The program was a simple timer with the following signature:

> 我记得在 20 世纪 90 年代曾为一套嵌入式实时系统编写过 C++程序。该程序是个简单的计时器，有如下签名：

```cpp
void Timer::ScheduleCommand(Command* theCommand, int milliseconds)
```

The idea was simple; the execute method of the Command would be executed in a new thread after the specified number of milliseconds. The problem was, how to test it.

> 想法很简单；到达指定毫秒数时，在一个新线程中执行 Command 的 excute 方法。问题在于如何测试它。

I cobbled together a simple driver program that listened to the keyboard. Every time a character was typed, it would schedule a command that would type the same character five seconds later. Then I tapped out a rhythmic melody on the keyboard and waited for that melody to replay on the screen five seconds later.

> 我随便写了个简单的驱动式程序，聆听来自键盘的动作。键盘输入一个字符时，它就安排 5 秒钟之后输出同样的字符。我输入了一句带节奏的歌词，然后等着 5 秒钟之后它在屏幕上重现出来。

“I … want-a-girl … just … like-the-girl-who-marr … ied … dear … old … dad.”

I actually sang that melody while typing the “.” key, and then I sang it again as the dots appeared on the screen.

> 在按下那些“.”键时，我真的在哼着那段旋律，当那些句点出现在屏幕上时，我又再哼了一次。

That was my test! Once I saw it work and demonstrated it to my colleagues, I threw the test code away.

> 那就是我的测试！我看到这法子可行，演示给同事们看，然后就把代码扔掉了。

As I said, our profession has come a long way. Nowadays I would write a test that made sure that every nook and cranny of that code worked as I expected it to. I would isolate my code from the operating system rather than just calling the standard timing functions. I would mock out those timing functions so that I had absolute control over the time. I would schedule commands that set boolean flags, and then I would step the time forward, watching those flags and ensuring that they went from false to true just as I changed the time to the right value.

> 如前文所述，我们的专业领域进步甚多。如今，我会编写测试，确保代码中每个犄角旮旯都如我所愿地工作。我会将代码和操作系统隔离开，而不是直接调用标准计时功能。我会伪造一套计时函数，这样就能全面控制时间。我会安排一些设置布尔值标识的命令，往前步进时间，查看这些标识，确保它们在我将时间调到正确值时由 false 变为 true。

Once I got a suite of tests to pass, I would make sure that those tests were convenient to run for anyone else who needed to work with the code. I would ensure that the tests and the code were checked in together into the same source package.

> 有了一套运行通过的测试，我会确保任何需要用到代码的人都能方便地使用这些测试。我会确保测试和代码一起签入同一个代码包。

Yes, we’ve come a long way; but we have farther to go. The Agile and TDD movements have encouraged many programmers to write automated unit tests, and more are joining their ranks every day. But in the mad rush to add testing to our discipline, many programmers have missed some of the more subtle, and important, points of writing good tests.

> 对，我们进步甚多；但还有很长的路要走。敏捷和 TDD 运动鼓舞了许多程序员编写自动化单元测试，每天还有更多人加入这个行列。但是，在争先恐后将测试加入规程中时，许多程序员遗漏了一些关于编写好测试的更细微但却重要的要点。

## 9.1 THE THREE LAWS OF TDD TDD 三定律

By now everyone knows that TDD asks us to write unit tests first, before we write production code. But that rule is just the tip of the iceberg. Consider the following three laws:

> 谁都知道 TDD 要求我们在编写生产代码前先编写单元测试。但这条规则只是冰山之巅。看看下列三定律：

http://doi.ieeecomputersociety.org/10.1109/MS.2007.85

First Law You may not write production code until you have written a failing unit test.

> 定律一 在编写不能通过的单元测试前，不可编写生产代码。

Second Law You may not write more of a unit test than is sufficient to fail, and not compiling is failing.

> 定律二 只可编写刚好无法通过的单元测试，不能编译也算不通过。

Third Law You may not write more production code than is sufficient to pass the currently failing test.

> 定律三 只可编写刚好足以通过当前失败测试的生产代码。

These three laws lock you into a cycle that is perhaps thirty seconds long. The tests and the production code are written together, with the tests just a few seconds ahead of the production code.

> 这三条定律将你限制在大概 30 秒一个的循环中。测试与生产代码一起编写，测试只比生产代码早写几秒钟。

If we work this way, we will write dozens of tests every day, hundreds of tests every month, and thousands of tests every year. If we work this way, those tests will cover virtually all of our production code. The sheer bulk of those tests, which can rival the size of the production code itself, can present a daunting management problem.

> 这样写程序，我们每天就会编写数十个测试，每个月编写数百个测试，每年编写数千个测试。这样写程序，测试将覆盖所有生产代码。测试代码量足以匹敌生产代码量，导致令人生畏的管理问题。

## 9.2 KEEPING TESTS CLEAN 保持测试整洁

Some years back I was asked to coach a team who had explicitly decided that their test code should not be maintained to the same standards of quality as their production code. They gave each other license to break the rules in their unit tests. “Quick and dirty” was the watchword. Their variables did not have to be well named, their test functions did not need to be short and descriptive. Their test code did not need to be well designed and thoughtfully partitioned. So long as the test code worked, and so long as it covered the production code, it was good enough.

> 几年前，有人请我去指导一个开发团队。那个团队认定，测试代码的维护不应遵循生产代码的质量标准。他们彼此默许在单元测试中破坏规矩。“速而不周”成了团队格言。变量命名不用好，测试函数不必短小和具有描述性。测试代码不必做良好设计和仔细划分。只要测试代码还能工作，只要还覆盖着生产代码，就足够好。

Some of you reading this might sympathize with that decision. Perhaps, long in the past, you wrote tests of the kind that I wrote for that Timer class. It’s a huge step from writing that kind of throw-away test, to writing a suite of automated unit tests. So, like the team I was coaching, you might decide that having dirty tests is better than having no tests.

> 有些读者可能会同意这种做法。或许，在很久以前，你也用过我为那个 Timer 类写测试的方法。从编写那种用后即扔的测试到编写全套自动化单元测试是一大进步。所以，就像那个我指导过的团队一样，你或许也会认为脏测试好过没测试。

What this team did not realize was that having dirty tests is equivalent to, if not worse than, having no tests. The problem is that tests must change as the production code evolves. The dirtier the tests, the harder they are to change. The more tangled the test code, the more likely it is that you will spend more time cramming new tests into the suite than it takes to write the new production code. As you modify the production code, old tests start to fail, and the mess in the test code makes it hard to get those tests to pass again. So the tests become viewed as an ever-increasing liability.

> 这个团队没有意识到的是，脏测试等同于——如果不是坏于的话 ——没测试。问题在于，测试必须随生产代码的演进而修改。测试越脏，就越难修改。测试代码越缠结，你就越有可能花更多时间塞进新测试，而不是编写新生产代码。修改生产代码后，旧测试就会开始失败，而测试代码中乱七八糟的东西将阻碍代码再次通过。于是，测试变得就像是不断翻番的债务。

From release to release the cost of maintaining my team’s test suite rose. Eventually it became the single biggest complaint among the developers. When managers asked why their estimates were getting so large, the developers blamed the tests. In the end they were forced to discard the test suite entirely.

随着版本递进，团队维护测试代码组的代价也在上升。最终，它变成了开发者最大的抱怨对象。当经理们问及为何超支如此巨大，开发者们就归咎于测试。最后，他们只能扔掉了整个测试代码组。

But, without a test suite they lost the ability to make sure that changes to their code base worked as expected. Without a test suite they could not ensure that changes to one part of their system did not break other parts of their system. So their defect rate began to rise. As the number of unintended defects rose, they started to fear making changes. They stopped cleaning their production code because they feared the changes would do more harm than good. Their production code began to rot. In the end they were left with no tests, tangled and bug-riddled production code, frustrated customers, and the feeling that their testing effort had failed them.

> 但是，没有了测试代码组，他们就失去了确保对代码的改动能如愿工作的能力。没有了测试代码组，他们就无法确保对系统某个部分的修改不会影响到系统的其他部分。故障率开始增加。随着并非出自有意的故障越来越多，他们开始害怕做改动。他们不再清理生产代码，因为他们害怕修改带来的损害多于收益。生产代码开始腐坏。最后，他们只剩下没有测试、纷乱而缺陷缠身的生产代码，沮丧的客户，还有对测试的失望。

In a way they were right. Their testing effort had failed them. But it was their decision to allow the tests to be messy that was the seed of that failure. Had they kept their tests clean, their testing effort would not have failed. I can say this with some certainty because I have participated in, and coached, many teams who have been successful with clean unit tests.

> 在某种意义上，他们说对了。测试的确让他们失望。不过是他们自己决定让测试变得乱七八糟的，而那正是失败的根源。如果他们保持测试整洁，测试就不会令他们失望。我可以拍着胸脯这么说，因为我曾经参与并指导了多个凭借整洁单元测试获得成功的团队。

The moral of the story is simple: Test code is just as important as production code. It is not a second-class citizen. It requires thought, design, and care. It must be kept as clean as production code.

> 故事的寓意很简单：测试代码和生产代码一样重要。它可不是二等公民。它需要被思考、被设计和被照料。它该像生产代码一般保持整洁。

### Tests Enable the -ilities 测试带来一切好处

If you don’t keep your tests clean, you will lose them. And without them, you lose the very thing that keeps your production code flexible. Yes, you read that correctly. It is unit tests that keep our code flexible, maintainable, and reusable. The reason is simple. If you have tests, you do not fear making changes to the code! Without tests every change is a possible bug. No matter how flexible your architecture is, no matter how nicely partitioned your design, without tests you will be reluctant to make changes because of the fear that you will introduce undetected bugs.

> 如果测试不能保持整洁，你就会失去它们。没有了测试，你就会失去保证生产代码可扩展的一切要素。你没看错。正是单元测试让你的代码可扩展、可维护、可复用。原因很简单。有了测试，你就不担心对代码的修改！没有测试，每次修改都可能带来缺陷。无论架构多有扩展性，无论设计划分得有多好，没有了测试，你就很难做改动，因为你担忧改动会引入不可预知的缺陷。

But with tests that fear virtually disappears. The higher your test coverage, the less your fear. You can make changes with near impunity to code that has a less than stellar architecture and a tangled and opaque design. Indeed, you can improve that architecture and design without fear!

> 有了测试，愁云一扫而空。测试覆盖率越高，你就越不担心。哪怕是对于那种架构并不优秀、设计晦涩纠缠的代码，你也能近乎没有后患地做修改。实际上，你能毫无顾虑地改进架构和设计！

So having an automated suite of unit tests that cover the production code is the key to keeping your design and architecture as clean as possible. Tests enable all the -ilities, because tests enable change.

> 所以，覆盖了生产代码的自动化单元测试程序组能尽可能地保持设计和架构的整洁。测试带来了一切好处，因为测试使改动变得可能。

So if your tests are dirty, then your ability to change your code is hampered, and you begin to lose the ability to improve the structure of that code. The dirtier your tests, the dirtier your code becomes. Eventually you lose the tests, and your code rots.

> 如果测试不干净，你改动自己代码的能力就有所牵制，而你也会开始失去改进代码结构的能力。测试越脏，代码就会变得越脏。最终，你丢失了测试，代码开始腐坏。

## 9.3 CLEAN TESTS 整洁的测试

What makes a clean test? Three things. Readability, readability, and readability. Readability is perhaps even more important in unit tests than it is in production code. What makes tests readable? The same thing that makes all code readable: clarity, simplicity, and density of expression. In a test you want to say a lot with as few expressions as possible.

> 整洁的测试有什么要素？有三个要素：可读性，可读性和可读性。在单元测试中，可读性甚至比在生产代码中还重要。测试如何才能做到可读？和其他代码中一样：明确，简洁，还有足够的表达力。在测试中，你要以尽可能少的文字表达大量内容。

Consider the code from FitNesse in Listing 9-1. These three tests are difficult to understand and can certainly be improved. First, there is a terrible amount of duplicate code [G5] in the repeated calls to addPage and assertSubString. More importantly, this code is just loaded with details that interfere with the expressiveness of the test.

> 来看看代码清单 9-1 中来自 FitNesse 的代码。这三个测试很难读懂，显然有改善空间。首先，其中有数量恐怖的重复代码[G5]调用 addPage 和 assertSubString。更重要的是，代码中充满了干扰测试表达力的细节。

Listing 9-1 SerializedPageResponderTest.java

> 代码清单 9-1 SerializedPageResponderTest.java

```java
public void testGetPageHieratchyAsXml() throws Exception {

    crawler.addPage(root, PathParser.parse("PageOne"));
    crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
    crawler.addPage(root, PathParser.parse("PageTwo"));

    request.setResource("root");
    request.addInput("type", "pages");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
            (SimpleResponse) responder.makeResponse(
                    new FitNesseContext(root), request);
    String xml = response.getContent();

    assertEquals("text / xml", response.getContentType());
    assertSubString(" < name > PageOne </name >", xml);
    assertSubString(" < name > PageTwo </name >", xml);
    assertSubString(" < name > ChildOne </name >", xml);
}

public void testGetPageHieratchyAsXmlDoesntContainSymbolicLinks() throws Exception {

    WikiPage pageOne = crawler.addPage(root, PathParser.parse("PageOne"));
    crawler.addPage(root, PathParser.parse("PageOne.ChildOne"));
    crawler.addPage(root, PathParser.parse("PageTwo"));

    PageData data = pageOne.getData();
    WikiPageProperties properties = data.getProperties();
    WikiPageProperty symLinks = properties.set(SymbolicPage.PROPERTY_NAME);
    symLinks.set("SymPage", "PageTwo");
    pageOne.commit(data);

    request.setResource("root");
    request.addInput("type", "pages");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
            (SimpleResponse) responder.makeResponse(
                    new FitNesseContext(root), request);
    String xml = response.getContent();

    assertEquals("text/xml", response.getContentType());
    assertSubString("<name>PageOne</name>", xml);
    assertSubString("<name>PageTwo</name>", xml);
    assertSubString("<name>ChildOne</name>", xml);
    assertNotSubString("SymPage", xml);
}

public void testGetDataAsHtml() throws Exception {
    crawler.addPage(root, PathParser.parse("TestPageOne"), "test page");

    request.setResource("TestPageOne");
    request.addInput("type", "data");
    Responder responder = new SerializedPageResponder();
    SimpleResponse response =
            (SimpleResponse) responder.makeResponse(
                    new FitNesseContext(root), request);
    String xml = response.getContent();

    assertEquals("text/xml", response.getContentType());
    assertSubString("test page", xml);
    assertSubString("<Test", xml);
}
```

For example, look at the PathParser calls. They transform strings into PagePath instances used by the crawlers. This transformation is completely irrelevant to the test at hand and serves only to obfuscate the intent. The details surrounding the creation of the responder and the gathering and casting of the response are also just noise. Then there’s the ham-handed way that the request URL is built from a resource and an argument. (I helped write this code, so I feel free to roundly criticize it.)

> 请看对 PathParser 的那些调用。它们将字符串转换为供爬虫使用的 PagePath 实体。转换与测试毫无关系，徒然混淆了代码的意图。与创建 responder 相关的细节，还有 response 的收集与转换也尽是噪声。此外还有从 resource 和参数构造请求 URL 的笨手段。（这些代码我有幸参与编写，所以可以敞开来批评。）

In the end, this code was not designed to be read. The poor reader is inundated with a swarm of details that must be understood before the tests make any real sense.

> 最终，这段代码不是设计来给人看的。可怜的读者淹没在细节的汪洋大海中，在真正用到测试之前，还得理解这些细节。

Now consider the improved tests in Listing 9-2. These tests do the exact same thing, but they have been refactored into a much cleaner and more explanatory form.

> 现在看看代码清单 9-2 中改进了的测试。这些测试还是做一样的事，不过已经被重构为更整洁和有表达力的形式。

Listing 9-2 SerializedPageResponderTest.java (refactored)

> 代码清单 9-2 SerializedPageResponderTest.java（重构后）

```java
public void testGetPageHierarchyAsXml() throws Exception {
    makePages("PageOne", "PageOne.ChildOne", "PageTwo");

    submitRequest("root", "type:pages");

    assertResponseIsXML();
    assertResponseContains(
            "<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>"
    );
}

public void testSymbolicLinksAreNotInXmlPageHierarchy() throws Exception {
    WikiPage page = makePage("PageOne");
    makePages("PageOne.ChildOne", "PageTwo");

    addLinkTo(page, "PageTwo", "SymPage");

    submitRequest("root", "type:pages");

    assertResponseIsXML();
    assertResponseContains(
            "<name>PageOne</name>", "<name>PageTwo</name>",
            "<name>ChildOne</name>"
    );
    assertResponseDoesNotContain("SymPage");
}

public void testGetDataAsXml() throws Exception {
    makePageWithContent("TestPageOne", "test page");

    submitRequest("TestPageOne", "type:data");

    assertResponseIsXML();
    assertResponseContains("test page", "<Test");
}
```

The BUILD-OPERATE-CHECK2 pattern is made obvious by the structure of these tests. Each of the tests is clearly split into three parts. The first part builds up the test data, the second part operates on that test data, and the third part checks that the operation yielded the expected results.

> 这些测试显然呈现了构造-操作-检验（BUILD-OPERATE-CHECK）模式。每个测试都清晰地拆分为三个环节。第一个环节构造测试数据，第二个环节操作测试数据，第三个部分检验操作是否得到期望的结果。

Notice that the vast majority of annoying detail has been eliminated. The tests get right to the point and use only the data types and functions that they truly need. Anyone who reads these tests should be able to work out what they do very quickly, without being misled or overwhelmed by details.

> 注意，那些恼人的细节大部分消失了。测试直达目的，只用到那些真正需要的数据类型和函数。读测试的人应该都能够很快搞清楚状况，不至于被细节误导或吓倒。

### 9.3.1 Domain-Specific Testing Language 面向特定领域的测试语言

The tests in Listing 9-2 demonstrate the technique of building a domain-specific language for your tests. Rather than using the APIs that programmers use to manipulate the system, we build up a set of functions and utilities that make use of those APIs and that make the tests more convenient to write and easier to read. These functions and utilities become a specialized API used by the tests. They are a testing language that programmers use to help themselves to write their tests and to help those who must read those tests later on.

> 代码清单 9-2 中的测试展示了为测试构造一种面向特定领域的语言的技巧。我们没有直接使用程序员用来对系统进行操作的 API，而是打造了一套包装这些 API 的函数和工具代码，这样就能更方便地编写测试，写出来的测试也更便于阅读。那正是一种测试语言，可以帮助程序员编写自己的测试，也可以帮助后来者阅读测试。

This testing API is not designed up front; rather it evolves from the continued refactoring of test code that has gotten too tainted by obfuscating detail. Just as you saw me refactor Listing 9-1 into Listing 9-2, so too will disciplined developers refactor their test code into more succinct and expressive forms.

> 这种测试 API 并非起初就设计出来，而是在对那些充满令人迷惑细节的测试代码进行后续重构时逐渐演进。如同你看见我将代码清单 9-1 重构为代码清单 9-2 一般，守规矩的开发者也将他们的测试代码重构为更简洁和具有表达力的形式。

### 9.3.2 A Dual Standard 双重标准

In one sense the team I mentioned at the beginning of this chapter had things right. The code within the testing API does have a different set of engineering standards than production code. It must still be simple, succinct, and expressive, but it need not be as efficient as production code. After all, it runs in a test environment, not a production environment, and those two environment have very different needs.

> 在某种意义上，本章开始处提到的那个团队的做法是正确的。测试 API 中的代码与生产代码相比，的确有一套不同的工程标准。测试代码应当简单、精悍、足具表达力，但它该和生产代码一般有效。毕竟它是在测试环境而非生产环境中运行，这两种环境有着截然不同的需求。

Consider the test in Listing 9-3. I wrote this test as part of an environment control system I was prototyping. Without going into the details you can tell that this test checks that the low temperature alarm, the heater, and the blower are all turned on when the temperature is “way too cold.”

> 请看代码清单 9-3 中的测试。在为某个环境控制系统设计原型时，我写了这个测试。无需深入细节，你就能说出该测试在“温度太低”时检验温度警报器、加热器和送风机是否全部打开。

Listing 9-3 EnvironmentControllerTest.java

> 代码清单 9-3 EnvironmentControllerTest.java

```java
@Test
public void turnOnLoTempAlarmAtThreashold() throws Exception {
    hw.setTemp(WAY_TOO_COLD);
    controller.tic();
    assertTrue(hw.heaterState());
    assertTrue(hw.blowerState());
    assertFalse(hw.coolerState());
    assertFalse(hw.hiTempAlarm());
    assertTrue(hw.loTempAlarm());
}
```

There are, of course, lots of details here. For example, what is that tic function all about? In fact, I’d rather you not worry about that while reading this test. I’d rather you just worry about whether you agree that the end state of the system is consistent with the temperature being “way too cold.”

> 当然，这里头也有许多细节。例如，tic 函数是做什么的？实际上，在读测试时你可以不用担心这些问题。你只需考虑是否同意系统最终状态是否与“温度太低”的情况相符。

Notice, as you read the test, that your eye needs to bounce back and forth between the name of the state being checked, and the sense of the state being checked. You see heaterState, and then your eyes glissade left to assertTrue. You see coolerState and your eyes must track left to assertFalse. This is tedious and unreliable. It makes the test hard to read.

> 当你阅读这个测试时，可以留意到自己的眼光得在被检验的状态的名称与状态的“意义”之间来回跳转。你看到 heaterState，眼光向左滑到 assertTrue。你看到 coolerState，眼光向左看 assertFalse。这个过程既乏味又不可靠。它让测试变得难以阅读。

I improved the reading of this test greatly by transforming it into Listing 9-4.

> 我大幅改进了测试的可读性，得到代码清单 9-4。

Listing 9-4 EnvironmentControllerTest.java (refactored)

> 代码清单 9-4 EnvironmentControllerTest.java（重构后）

```java
@Test
public void turnOnLoTempAlarmAtThreshold() throws Exception {
    wayTooCold();
    assertEquals("HBchL", hw.getState());
}
```

Of course I hid the detail of the tic function by creating a wayTooCold function. But the thing to note is the strange string in the assertEquals. Upper case means “on,” lower case means “off,” and the letters are always in the following order: {heater, blower, cooler, hi-temp-alarm, lo-temp-alarm}.

> 当然，我创建了一个 wayTooCold 函数，隐藏了 tic 函数的细节。不过要注意的是 assertEquals 中的那个奇怪的字符串。大写表示“打开”，小写表示“关闭”，那些字符遵循以下次序：{heater, blower, cooler, hitemp-alarm, lo-temp-alarm}。

Even though this is close to a violation of the rule about mental mapping,3 it seems appropriate in this case. Notice, once you know the meaning, your eyes glide across that string and you can quickly interpret the results. Reading the test becomes almost a pleasure. Just take a look at Listing 9-5 and see how easy it is to understand these tests.

> 尽管这破坏了思维映射的规则，看来它在这种情况下还是适用的。只要你明白其含义，你就能一眼看到那个字符串，迅速译解出结果。

Listing 9-5 EnvironmentControllerTest.java (bigger selection)

> 代码清单 9-5 EnvironmentControllerTest.java（扩展到更大范围）

```java
@Test
public void turnOnCoolerAndBlowerIfTooHot() throws Exception {
    tooHot();
    assertEquals("hBChl", hw.getState());
}

@Test
public void turnOnHeaterAndBlowerIfTooCold() throws Exception {
    tooCold();
    assertEquals("HBchl", hw.getState());
}

@Test
public void turnOnHiTempAlarmAtThreshold() throws Exception {
    wayTooHot();
    assertEquals("hBCHl", hw.getState());
}

@Test
public void turnOnLoTempAlarmAtThreshold() throws Exception {
    wayTooCold();
    assertEquals("HBchL", hw.getState());
}
```

The getState function is shown in Listing 9-6. Notice that this is not very efficient code. To make it efficient, I probably should have used a StringBuffer.

> 代码清单 9-6 中给出了 getState 函数。注意，代码效率不是非常高。要提升效率，可能应该使用 StringBuffer。

Listing 9-6 MockControlHardware.java

> 代码清单 9-6 MockControlHardware.java

```java
public String getState() {
    String state = "";
    state += heater ? "H" : "h";
    state += blower ? "B" : "b";
    state += cooler ? "C" : "c";
    state += hiTempAlarm ? "H" : "h";
    state += loTempAlarm ? "L" : "l";
    return state;
}
```

StringBuffers are a bit ugly. Even in production code I will avoid them if the cost is small; and you could argue that the cost of the code in Listing 9-6 is very small. However, this application is clearly an embedded real-time system, and it is likely that computer and memory resources are very constrained. The test environment, however, is not likely to be constrained at all.

> StringBuffer 有点丑陋。即便在生产代码中，假使代价较小，我都会避免使用 StringBuffer；而且你可以看到，清单 9-6 中代码的代价的确很小。这套应用显然是嵌入式实时系统，计算机和内存资源都很有限。不过，测试环境大概完全不必做限制。

That is the nature of the dual standard. There are things that you might never do in a production environment that are perfectly fine in a test environment. Usually they involve issues of memory or CPU efficiency. But they never involve issues of cleanliness.

> 这就是双重标准。有些事你大概永远不会在生产环境中做，而在测试环境中做却完全没问题。通常这关乎内存或 CPU 效率的问题，不过却永远不会与整洁有关。

## 9.4 ONE ASSERT PER TEST 每个测试一个断言

There is a school of thought4 that says that every test function in a JUnit test should have one and only one assert statement. This rule may seem draconian, but the advantage can be seen in Listing 9-5. Those tests come to a single conclusion that is quick and easy to understand.

> 有个流派认为，JUnit 中每个测试函数都应该有且只有一个断言语句。这条规则看似过于苛求，但其好处却可以在代码清单 9-5 中看到。这些测试都归结为一个可快速方便地理解的结论。

But what about Listing 9-2? It seems unreasonable that we could somehow easily merge the assertion that the output is XML and that it contains certain substrings. However, we can break the test into two separate tests, each with its own particular assertion, as shown in Listing 9-7.

> 代码清单 9-2 又如何？我们能将关于输出是 XML 的断言与输出包含某些子字符串的断言轻易地组合到一起，不过这样做看来毫无道理。然而，我们可以将测试分解为两个单独的测试，每个都有自己的断言，如代码清单 9-7 所示。

Listing 9-7 SerializedPageResponderTest.java (Single Assert)

> 代码清单 9-7 SerializedPageResponderTest.java（单个断言的版本）

```java
public void testGetPageHierarchyAsXml() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");

    whenRequestIsIssued("root", "type:pages");

    thenResponseShouldBeXML();
}

public void testGetPageHierarchyHasRightTags() throws Exception {
    givenPages("PageOne", "PageOne.ChildOne", "PageTwo");

    whenRequestIsIssued("root", "type:pages");

    thenResponseShouldContain("<name>PageOne</name>", "<name>PageTwo</name>", "<name>ChildOne</name>");
}
```

Notice that I have changed the names of the functions to use the common given-when-then5 convention. This makes the tests even easier to read. Unfortunately, splitting the tests as shown results in a lot of duplicate code.

> 注意，我修改了那些函数的名称，以符合 given-when-then 约定。这让测试更易阅读。不幸的是，如此分解测试，导致了许多重复代码的出现。

We can eliminate the duplication by using the TEMPLATE METHOD6 pattern and putting the given/when parts in the base class, and the then parts in different derivatives. Or we could create a completely separate test class and put the given and when parts in the @Before function, and the when parts in each @Test function. But this seems like too much mechanism for such a minor issue. In the end, I prefer the multiple asserts in Listing 9-2.

> 可以利用模板方法（TEMPLATE METHOD）模式，将 given/when 部分放到基类中，将 then 部分放到派生类中，消除代码重复问题。或者，我们也可以创建一个完整的单独测试类，把 given 和 when 部分放到@Before 函数中，把 when 部分放到每个@Test 函数中。但对于这个小问题，这看来有点太机械。最后，我还是保留了代码清单 9-2 那种多个断言的形式。

I think the single assert rule is a good guideline.7 I usually try to create a domain-specific testing language that supports it, as in Listing 9-5. But I am not afraid to put more than one assert in a test. I think the best thing we can say is that the number of asserts in a test ought to be minimized.

> 我认为，单个断言是个好准则。我通常都会创建支持这条准则的特定领域测试语言，如代码清单 9-5 所示。不过，我也不害怕在单个测试中放入一个以上断言。我认为，最好的说法是单个测试中的断言数量应该最小化。

### Single Concept per Test 每个测试一个概念

Perhaps a better rule is that we want to test a single concept in each test function. We don’t want long test functions that go testing one miscellaneous thing after another. Listing 9-8 is an example of such a test. This test should be split up into three independent tests because it tests three independent things. Merging them all together into the same function forces the reader to figure out why each section is there and what is being tested by that section.

> 更好一些的规则或许是每个测试函数中只测试一个概念。我们不想要超长的测试函数，测试完这个又测试那个。代码清单 9-8 就是那样一种测试的例子。这个测试应当拆解为 3 个单独测试，因为它测试了 3 件不同的事。把三者混到一起，读者就不得不猜想每段代码出现的理由，以及那段代码到底要测试什么。

Listing 9-8

> 代码清单 9-8

```java
/**
 * Miscellaneous tests for the addMonths() method.
 */
public void testAddMonths() {
    SerialDate d1 = SerialDate.createInstance(31, 5, 2004);

    SerialDate d2 = SerialDate.addMonths(1, d1);
    assertEquals(30, d2.getDayOfMonth());
    assertEquals(6, d2.getMonth());
    assertEquals(2004, d2.getYYYY());

    SerialDate d3 = SerialDate.addMonths(2, d1);
    assertEquals(31, d3.getDayOfMonth());
    assertEquals(7, d3.getMonth());
    assertEquals(2004, d3.getYYYY());

    SerialDate d4 = SerialDate.addMonths(1, SerialDate.addMonths(1, d1));
    assertEquals(30, d4.getDayOfMonth());
    assertEquals(7, d4.getMonth());
    assertEquals(2004, d4.getYYYY());
}
```

The three test functions probably ought to be like this:

> 这三个测试函数大概应该像这个样子：

- Given the last day of a month with 31 days (like May):

> 对于某个有 31 天的月份的最后一天（如五月）：

1. When you add one month, such that the last day of that month is the 30th (like June), then the date should be the 30th of that month, not the 31st.

> （1） 增加一个该月份最末一天为 30 日（如六月）的月份时，日期应该是该月的 30 日而非 31 日。

2. When you add two months to that date, such that the final month has 31 days, then the date should be the 31st.

> （2） 增加最末月有 31 天的两个月时，日期应该是 31 日。

- Given the last day of a month with 30 days in it (like June):

> 对于某个有 30 天的月份的最后一天（如六月）：

3. When you add one month such that the last day of that month has 31 days, then the date should be the 30th, not the 31st.

> 增加一个有 31 天的月份时，日期应该是 30 日而非 31 日。

Stated like this, you can see that there is a general rule hiding amidst the miscellaneous tests. When you increment the month, the date can be no greater than the last day of the month. This implies that incrementing the month on February 28th should yield March 28th. That test is missing and would be a useful test to write.

> 这样一来，你可以看到，在这些混杂的测试当中，隐藏有一条普遍规则。增加月份数时，日期不能大于该月份的最末一天。这意味着在 2 月 28 日增加月份数，就会得到 3 月 28 日。而这个测试应该有用，但被遗漏了。

So it’s not the multiple asserts in each section of Listing 9-8 that causes the problem. Rather it is the fact that there is more than one concept being tested. So probably the best rule is that you should minimize the number of asserts per concept and test just one concept per test function.

> 并非是代码清单 9-8 中每个段落的多重断言导致问题。问题在于，有多个概念被测试，所以，最佳规则也许是应该尽可能减少每个概念的断言数量，每个测试函数只测试一个概念。

## 9.5 F.I.R.S.T.

Clean tests follow five other rules that form the above acronym:

> 整洁的测试还遵循以下 5 条规则，这 5 条规则的首字母构成了本节标题：

Fast Tests should be fast. They should run quickly. When tests run slow, you won’t want to run them frequently. If you don’t run them frequently, you won’t find problems early enough to fix them easily. You won’t feel as free to clean up the code. Eventually the code will begin to rot.

> 快速（Fast）测试应该够快。测试应该能快速运行。测试运行缓慢，你就不会想要频繁地运行它。如果你不频繁运行测试，就不能尽早发现问题，也无法轻易修正，从而也不能轻而易举地清理代码。最终，代码就会腐坏。

Independent Tests should not depend on each other. One test should not set up the conditions for the next test. You should be able to run each test independently and run the tests in any order you like. When tests depend on each other, then the first one to fail causes a cascade of downstream failures, making diagnosis difficult and hiding downstream defects.

> 独立（Independent）测试应该相互独立。某个测试不应为下一个测试设定条件。你应该可以单独运行每个测试，及以任何顺序运行测试。当测试互相依赖时，头一个没通过就会导致一连串的测试失败，使问题诊断变得困难，隐藏了下级错误。

Repeatable Tests should be repeatable in any environment. You should be able to run the tests in the production environment, in the QA environment, and on your laptop while riding home on the train without a network. If your tests aren’t repeatable in any environment, then you’ll always have an excuse for why they fail. You’ll also find yourself unable to run the tests when the environment isn’t available.

> 可重复（Repeatable）测试应当可在任何环境中重复通过。你应该能够在生产环境、质检环境中运行测试，也能够在无网络的列车上用笔记本电脑运行测试。如果测试不能在任意环境中重复，你就总会有个解释其失败的接口。当环境条件不具备时，你也会无法运行测试。

Self-Validating The tests should have a boolean output. Either they pass or fail. You should not have to read through a log file to tell whether the tests pass. You should not have to manually compare two different text files to see whether the tests pass. If the tests aren’t self-validating, then failure can become subjective and running the tests can require a long manual evaluation.

> 自足验证（Self-Validating）测试应该有布尔值输出。无论是通过或失败，你不应该查看日志文件来确认测试是否通过。你不应该手工对比两个不同文本文件来确认测试是否通过。如果测试不能自足验证，对失败的判断就会变得依赖主观，而运行测试也需要更长的手工操作时间。

Timely The tests need to be written in a timely fashion. Unit tests should be written just before the production code that makes them pass. If you write tests after the production code, then you may find the production code to be hard to test. You may decide that some production code is too hard to test. You may not design the production code to be testable.

> 及时（Timely）测试应及时编写。单元测试应该恰好在使其通过的生产代码之前编写。如果在编写生产代码之后编写测试，你会发现生产代码难以测试。你可能会认为某些生产代码本身难以测试。你可能不会去设计可测试的代码。

## 9.6 CONCLUSION 小结

We have barely scratched the surface of this topic. Indeed, I think an entire book could be written about clean tests. Tests are as important to the health of a project as the production code is. Perhaps they are even more important, because tests preserve and enhance the flexibility, maintainability, and reusability of the production code. So keep your tests constantly clean. Work to make them expressive and succinct. Invent testing APIs that act as domain-specific language that helps you write the tests.

> 我们只是触及了这个话题的表面。实际上，我认为应该为整洁的测试写上一整本书。对于项目的健康度，测试盒生产代码同等重要。或许测试更为重要，因为它保证和增强了生产代码的可扩展性、可维护性和可复用性。所以，保持测试整洁吧。让测试具有表达力并短小精悍。发明作为面向特定领域语言的测试 API，帮助自己编写测试。

If you let the tests rot, then your code will rot too. Keep your tests clean.

> 如果你坐视测试腐坏，那么代码也会跟着腐坏。保持测试整洁吧。
