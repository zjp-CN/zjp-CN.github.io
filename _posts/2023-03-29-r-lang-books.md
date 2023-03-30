---
title: 【R 语言】在线书籍
date: 2023-03-29 15:49:44 +0800
categories: [R 语言]
tags: [R 语言, 书籍]
---

> 学习路径： [RStudio Education](https://education.rstudio.com/learn/)
>
> RStudio 团队成员写的书籍：[books](https://www.rstudio.com/resources/books/)
{: .prompt-tip }

# Hadley Wickham

Hadley Wickham 大佬的博客：<https://hadley.nz/>

* with Garrett Grolemund, the place to start if you want to learn how to do data science with R.
  * [R for Data Science](http://r4ds.had.co.nz/)：第一版
  * [R for Data Science (2e)](https://r4ds.hadley.nz/)：第二版（2023 年中预计完成）
* [ggplot2: elegant graphics for data analysis](https://ggplot2-book.org) shows you how to use ggplot2 to create graphics that help you understand your data.
* [Advanced R](http://adv-r.hadley.nz/) helps you master R as a programming language, teaching you what makes R tick.
* [R packages](http://r-pkgs.org/) teaches good software engineering practices for R, using packages for bundling, documenting, and testing your code.

# 其他优秀书籍

* [R Graphics Cookbook, 2nd edition](https://r-graphics.org/)：作者 [Winston Chang](https://github.com/wch/rcookbook)
* [Advanced R Solutions](https://advanced-r-solutions.rbind.io/)：作者 Malte Grosser，Hadley 的 《Advanced R》 书籍习题解答
* [Hands-On Programming with R](https://rstudio-education.github.io/hopr/)：作者 Garrett Grolemund

# ggplot2 书籍

见 <https://ggplot2.tidyverse.org/#learning-ggplot2>

> If you are new to ggplot2 you are better off starting with a systematic introduction, rather than trying to learn from reading individual documentation pages. Currently, there are three good places to start:
> 
> 1.  The [Data Visualisation](https://r4ds.had.co.nz/data-visualisation.html) and [Graphics for communication](https://r4ds.had.co.nz/graphics-for-communication.html) chapters in [R for Data Science](https://r4ds.had.co.nz). R for Data Science is designed to give you a comprehensive introduction to the [tidyverse](https://www.tidyverse.org), and these two chapters will get you up to speed with the essentials of ggplot2 as quickly as possible.
>     
> 2.  If you’d like to take an online course, try [Data Visualization in R With ggplot2](https://learning.oreilly.com/videos/data-visualization-in/9781491963661/) by Kara Woo.
>     
> 3.  If you’d like to follow a webinar, try [Plotting Anything with ggplot2](https://youtu.be/h29g21z0a68) by Thomas Lin Pedersen.
>     
> 4.  If you want to dive into making common graphics as quickly as possible, I recommend [The R Graphics Cookbook](https://r-graphics.org) by Winston Chang. It provides a set of recipes to solve common graphics problems.
>     
> 
> If you’ve mastered the basics and want to learn more, read [ggplot2: Elegant Graphics for Data Analysis](https://ggplot2-book.org). It describes the theoretical underpinnings of ggplot2 and shows you how all the pieces fit together. This book helps you understand the theory that underpins ggplot2, and will help you create new types of graphics specifically tailored to your needs.

# Cheat Sheet

* RStudio 的 cheatsheets 仓库：<https://github.com/rstudio/cheatsheets>
  * 官网：<https://www.rstudio.com/resources/cheatsheets/>
* ggplot2：<https://github.com/rstudio/cheatsheets/blob/main/data-visualization.pdf>


# 小技巧

* 重置 DataFrame 的索引：`row.names(df) <- NULL`
