---
weight: 3
title: 3. 函数
---

# 第 3 章 函数

![](/cc/figures/ch3/3_1fig_martin.jpg)

In the early days of programming we composed our systems of routines and subroutines. Then, in the era of Fortran and PL/1 we composed our systems of programs, subprograms, and functions. Nowadays only the function survives from those early days. Functions are the first line of organization in any program. Writing them well is the topic of this chapter.

> 在编程的早年岁月，系统由程序和子程序组成。后来，在 Fortran 和 PL/1 的年代，系统由程序、子程序和函数组成。如今，只有函数存活下来。函数是所有程序中的第一组代码。本章将讨论如何写好函数。

Consider the code in Listing 3-1. It’s hard to find a long function in FitNesse,1 but after a bit of searching I came across this one. Not only is it long, but it’s got duplicated code, lots of odd strings, and many strange and inobvious data types and APIs. See how much sense you can make of it in the next three minutes.

> 请看代码清单 3-1。在 FitNesse 中，很难找到长函数，不过我还是搜寻到一个。它不光长，而且代码也很复杂，有大量字符串、怪异而不显见的数据类型和 API。花 3 分钟时间，看能读懂多少？

Listing 3-1 HtmlUtil.java (FitNesse 20070619)

> 代码清单 3-1 HtmlUtil.java（FitNesse 20070619）

```java
public static String testableHtml(
        PageData pageData,
        boolean includeSuiteSetup
) throws Exception {
    WikiPage wikiPage = pageData.getWikiPage();
    StringBuffer buffer = new StringBuffer();
    if (pageData.hasAttribute("Test")) {
        if (includeSuiteSetup) {
            WikiPage suiteSetup =
                    PageCrawlerImpl.getInheritedPage(
                            SuiteResponder.SUITE_SETUP_NAME, wikiPage
                    );
            if (suiteSetup != null) {
                WikiPagePath pagePath =
                        suiteSetup.getPageCrawler().getFullPath(suiteSetup);
                String pagePathName = PathParser.render(pagePath);
                buffer.append("!include -setup .")
                        .append(pagePathName)
                        .append("\n");
            }
        }
        WikiPage setup =
                PageCrawlerImpl.getInheritedPage("SetUp", wikiPage);
        if (setup != null) {
            WikiPagePath setupPath =
                    wikiPage.getPageCrawler().getFullPath(setup);
            String setupPathName = PathParser.render(setupPath);
            buffer.append("!include -setup .")
                    .append(setupPathName)
                    .append("\n");
        }
    }
    buffer.append(pageData.getContent());
    if (pageData.hasAttribute("Test")) {
        WikiPage teardown =
                PageCrawlerImpl.getInheritedPage("TearDown", wikiPage);
        if (teardown != null) {
            WikiPagePath tearDownPath =
                    wikiPage.getPageCrawler().getFullPath(teardown);
            String tearDownPathName = PathParser.render(tearDownPath);
            buffer.append("\n")
                    .append("!include -teardown .")
                    .append(tearDownPathName)
                    .append("\n");
        }
        if (includeSuiteSetup) {
            WikiPage suiteTeardown =
                    PageCrawlerImpl.getInheritedPage(
                            SuiteResponder.SUITE_TEARDOWN_NAME,
                            wikiPage
                    );
            if (suiteTeardown != null) {
                WikiPagePath pagePath =
                        suiteTeardown.getPageCrawler().getFullPath(suiteTeardown);
                String pagePathName = PathParser.render(pagePath);
                buffer.append("!include -teardown .")
                        .append(pagePathName)
                        .append("\n");
            }
        }
    }
    pageData.setContent(buffer.toString());
    return pageData.getHtml();
}
```

Do you understand the function after three minutes of study? Probably not. There’s too much going on in there at too many different levels of abstraction. There are strange strings and odd function calls mixed in with doubly nested if statements controlled by flags.

> 搞懂这个函数了吗？大概没有。有太多事发生，有太多不同层级的抽象。奇怪的字符串和函数调用，混以双重嵌套、用标识来控制的 if 语句等，不一而足。

However, with just a few simple method extractions, some renaming, and a little restructuring, I was able to capture the intent of the function in the nine lines of Listing 3-2. See whether you can understand that in the next 3 minutes.

> 不过，只要做几个简单的方法抽离和重命名操作，加上一点点重构，就能在 9 行代码之内搞掂（如代码清单 3-2 所示）。用 3 分钟阅读以下代码，看你能理解吗？

Listing 3-2 HtmlUtil.java (refactored)

> 代码清单 3-2 HtmlUtil.java（重构之后）

```java
public static String renderPageWithSetupsAndTeardowns(
        PageData pageData, boolean isSuite
) throws Exception {
    boolean isTestPage = pageData.hasAttribute("Test");
    if (isTestPage) {
        WikiPage testPage = pageData.getWikiPage();
        StringBuffer newPageContent = new StringBuffer();
        includeSetupPages(testPage, newPageContent, isSuite);
        newPageContent.append(pageData.getContent());
        includeTeardownPages(testPage, newPageContent, isSuite);
        pageData.setContent(newPageContent.toString());
    }

    return pageData.getHtml();
}
```

Unless you are a student of FitNesse, you probably don’t understand all the details. Still, you probably understand that this function performs the inclusion of some setup and teardown pages into a test page and then renders that page into HTML. If you are familiar with JUnit,2 you probably realize that this function belongs to some kind of Web-based testing framework. And, of course, that is correct. Divining that information from Listing 3-2 is pretty easy, but it’s pretty well obscured by Listing 3-1.

> 除非你正在研究 FitNesse，否则就理解不了所有细节。不过，你大概能明白，该函数包含把一些设置和拆解页放入一个测试页面，再渲染为 HTML 的操作。如果你熟悉 JUnit，或许会想到，该函数归属于某个基于 Web 的测试框架。而且，这当然没错。从代码清单 3-2 中获得信息很容易，而代码清单 3-1 则晦涩难明。

So what is it that makes a function like Listing 3-2 easy to read and understand? How can we make a function communicate its intent? What attributes can we give our functions that will allow a casual reader to intuit the kind of program they live inside?

> 是什么让代码清单 3-2 易于阅读和理解？怎么才能让函数表达其意图？该给函数赋予哪些属性，好让读者一看就明白函数是属于怎样的程序？