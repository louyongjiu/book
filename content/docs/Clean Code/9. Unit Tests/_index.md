---
weight: 9
title: "9. 单元测试"
---

# 第 9 章 单元测试

![](/cc/figures/ch9/9_1fig_martin.jpg)

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