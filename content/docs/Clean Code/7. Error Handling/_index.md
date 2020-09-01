---
weight: 7
title: "7. 错误处理"
---

# 第 7 章 错误处理

by Michael Feathers

![](/cc/figures/ch7/103fig01.jpg)

It might seem odd to have a section about error handling in a book about clean code. Error handling is just one of those things that we all have to do when we program. Input can be abnormal and devices can fail. In short, things can go wrong, and when they do, we as programmers are responsible for making sure that our code does what it needs to do.

> 在一本有关整洁代码的书中，居然有讨论错误处理的章节，看起来有些突兀。错误处理只不过是编程时必须要做的事之一。输入可能出现异常，设备可能失效。简言之，可能会出错，当错误发生时，程序员就有责任确保代码照常工作。

The connection to clean code, however, should be clear. Many code bases are completely dominated by error handling. When I say dominated, I don’t mean that error handling is all that they do. I mean that it is nearly impossible to see what the code does because of all of the scattered error handling. Error handling is important, but if it obscures logic, it’s wrong.

> 然而，应该弄清楚错误处理与整洁代码的关系。许多程序完全由错误处理所占据。所谓占据，并不是说错误处理就是全部。我的意思是几乎无法看明白代码所做的事，因为到处都是凌乱的错误处理代码。错误处理很重要，但如果它搞乱了代码逻辑，就是错误的做法。

In this chapter I’ll outline a number of techniques and considerations that you can use to write code that is both clean and robust—code that handles errors with grace and style.

> 在本章中，我将概要列出编写既整洁又强固的代码——雅致地处理错误代码的一些技巧和思路。