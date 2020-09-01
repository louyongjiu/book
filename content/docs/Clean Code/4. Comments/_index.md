---
weight: 4
title: "4. 注释"
---

# 第 4 章 注释

![](/cc/figures/ch4/4_1fig_martin.jpg)

“Don’t comment bad code—rewrite it.”—Brian W. Kernighan and P. J. Plaugher1

> “别给糟糕的代码加注释——重新写吧。”——Brian W. Kernighan 与 P. J.

Nothing can be quite so helpful as a well-placed comment. Nothing can clutter up a module more than frivolous dogmatic comments. Nothing can be quite so damaging as an old crufty comment that propagates lies and misinformation.

> 什么也比不上放置良好的注释来得有用。什么也不会比乱七八糟的注释更有本事搞乱一个模块。什么也不会比陈旧、提供错误信息的注释更有破坏性。

Comments are not like Schindler’s List. They are not “pure good.” Indeed, comments are, at best, a necessary evil. If our programming languages were expressive enough, or if we had the talent to subtly wield those languages to express our intent, we would not need comments very much—perhaps not at all.

> 注释并不像辛德勒的名单。它们并不“纯然地好”。实际上，注释最多也就是一种必须的恶。若编程语言足够有表达力，或者我们长于用这些语言来表达意图，就不那么需要注释——也许根本不需要。

The proper use of comments is to compensate for our failure to express ourself in code. Note that I used the word failure. I meant it. Comments are always failures. We must have them because we cannot always figure out how to express ourselves without them, but their use is not a cause for celebration.

> 注释的恰当用法是弥补我们在用代码表达意图时遭遇的失败。注意，我用了“失败”一词。我是说真的。注释总是一种失败。我们总无法找到不用注释就能表达自我的方法，所以总要有注释，这并不值得庆贺。

So when you find yourself in a position where you need to write a comment, think it through and see whether there isn’t some way to turn the tables and express yourself in code. Every time you express yourself in code, you should pat yourself on the back. Every time you write a comment, you should grimace and feel the failure of your ability of expression.

> 如果你发现自己需要写注释，再想想看是否有办法翻盘，用代码来表达。每次用代码表达，你都该夸奖一下自己。每次写注释，你都该做个鬼脸，感受自己在表达能力上的失败。

Why am I so down on comments? Because they lie. Not always, and not intentionally, but too often. The older a comment is, and the farther away it is from the code it describes, the more likely it is to be just plain wrong. The reason is simple. Programmers can’t realistically maintain them.

> 我为什么要极力贬低注释？因为注释会撒谎。也不是说总是如此或有意如此，但出现得实在太频繁。注释存在的时间越久，就离其所描述的代码越远，越来越变得全然错误。原因很简单。程序员不能坚持维护注释。

Code changes and evolves. Chunks of it move from here to there. Those chunks bifurcate and reproduce and come together again to form chimeras. Unfortunately the comments don’t always follow them—can’t always follow them. And all too often the comments get separated from the code they describe and become orphaned blurbs of ever-decreasing accuracy. For example, look what has happened to this comment and the line it was intended to describe:

> 代码在变动，在演化。从这里移到那里。彼此分离、重造又合到一处。很不幸，注释并不总是随之变动——不能总是跟着走。注释常常会与其所描述的代码分隔开来，孑然飘零，越来越不准确。例如，看看以下注释以及它本来要描述的代码行变成了什么样子：

```java
MockRequest request;
private final String HTTP_DATE_REGEXP =
        "[SMTWF][a-z]{2}\\,\\s[0-9]{2}\\s[JFMASOND][a-z]{2}\\s" +
                "[0-9]{4}\\s[0-9]{2}\\:[0-9]{2}\\:[0-9]{2}\\sGMT";
private Response response;
private FitNesseContext context;
private FileResponder responder;
private Locale saveLocale;
// Example: "Tue, 02 Apr 2003 22:18:49 GMT"
```

Other instance variables that were probably added later were interposed between the HTTP_DATE_REGEXP constant and it’s explanatory comment.

> 在 HTTP_DATE_REGEXP 常量及其注释之间，有可能插入其他实体变量。

It is possible to make the point that programmers should be disciplined enough to keep the comments in a high state of repair, relevance, and accuracy. I agree, they should. But I would rather that energy go toward making the code so clear and expressive that it does not need the comments in the first place.

> 程序员应当负责将注释保持在可维护、有关联、精确的高度。我同意这种说法。但我更主张把力气用在写清楚代码上，直接保证无须编写注释。

Inaccurate comments are far worse than no comments at all. They delude and mislead. They set expectations that will never be fulfilled. They lay down old rules that need not, or should not, be followed any longer.

> 不准确的注释要比没注释坏得多。它们满口胡言。它们预期的东西永不能实现。它们设定了无需也不应再遵循的旧规则。

Truth can only be found in one place: the code. Only the code can truly tell you what it does. It is the only source of truly accurate information. Therefore, though comments are sometimes necessary, we will expend significant energy to minimize them.

> 真实只在一处地方有：代码。只有代码能忠实地告诉你它做的事。那是唯一真正准确的信息来源。所以，尽管有时也需要注释，我们也该多花心思尽量减少注释量。