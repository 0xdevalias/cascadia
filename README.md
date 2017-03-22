# cascadia

[![MIT License](http://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Go Report Card](https://goreportcard.com/badge/github.com/suntong/cascadia)](https://goreportcard.com/report/github.com/suntong/cascadia)
[ ![Codeship Status](https://codeship.com/projects/7fbac590-a3dd-0134-4b89-26c19bdf8358/status?branch=master)](https://codeship.com/projects/190387)

The [Go Cascadia package](https://github.com/andybalholm/cascadia) implements CSS selectors for html. This is the command line tool, started as a thin wrapper around that package, but growing into a better tool to test CSS selectors without writing Go code:

```
$ cascadia
cascadia wrapper
built on 2017-03-22

Command line interface to go cascadia CSS selectors package

Options:

  -h, --help            display help information
  -i, --in             *The html/xml file to read from (or stdin)
  -o, --out            *The output file (or stdout)
  -c, --css            *CSS selectors
  -p, --piece           sub CSS selectors within -css to split that block up into pieces
  -d, --delimiter[=     ]   delimiter for pieces csv output
```

Its output has two modes, _single selection mode_ and _block selection mode_, depending on whether the `--piece` parameter is given on the command line.

- The single selection mode will output the selection as HTML source, while
- The block selection mode will output HTML text in a tsv/csv table form

For details about the concept of block and pieces, check out [andrew-d/goscrape](https://github.com/andrew-d/goscrape) (in fact, `cascadia` was initially developed just for it, so that I don't need to tweak Go code, build & run it just to test out the block and pieces selectors). Here is the exception:

- Inside each page, there's 1 or more *blocks* - some logical method of splitting up a page into subcomponents.
- Inside each block, you define some number of *pieces* of data that you wish
  to extract.  Each piece consists of a name, a selector, and what data to
  extract from the current block.

This all sounds rather complicated, but in practice it's quite simple. See the next section for details.


## Examples

### Single selection mode

All the three `-i -o -c` options are required. By default it reads from `stdin` and output to `stdout`:

```sh
$ echo '<input type="radio" name="Sex" value="F" />' | tee /tmp/cascadia.xml | cascadia -i -o -c 'input[name=Sex][value=M]'
0 elements for 'input[name=Sex][value=M]':
```

Either the input or the output can be followed by a file name:


```sh
$ cascadia -i /tmp/cascadia.xml -o -c 'input[name=Sex][value=F]'
1 elements for 'input[name=Sex][value=F]':
<input type="radio" name="Sex" value="F"/>
```

Of course, any number of selections allowed:

```sh
$ echo '<table border="0" cellpadding="0" cellspacing="0" style="table-layout: fixed; width: 100%; border: 0 dashed; border-color: #FFFFFF"><tr style="height:64px">aaa</tr></table>' | cascadia -i -o -c 'table[border="0"][cellpadding="0"][cellspacing="0"], tr[style=height\:64px]'
2 elements for 'table[border="0"][cellpadding="0"][cellspacing="0"], tr[style=height\:64px]':
<table border="0" cellpadding="0" cellspacing="0" style="table-layout: fixed; width: 100%; border: 0 dashed; border-color: #FFFFFF"><tbody><tr style="height:64px"></tr></tbody></table>
<tr style="height:64px"></tr>
```

### Block selection mode

First, as the single selection mode will output the selection as HTML source, so if you want HTML text instead, then you can make use of the block selection mode. 

```sh
$ echo '<div class="container"><p align="justify"><b>Name: </b>John Doe</p></div>' | tee /tmp/cascadia.xml | cascadia -i -o -c 'div > p'
1 elements for 'div > p':
<p align="justify"><b>Name: </b>John Doe</p>

$ cat /tmp/cascadia.xml | cascadia -i -o -c 'div' --piece SelText='p'
SelText
Name: John Doe
```

However, the real power of _block selection mode_ resides in its capability of producing tsv/csv tables without any go programming:


```
$ curl --silent https://news.ycombinator.com | cascadia -i -o -c 'tr.athing' -p No=span.rank -p Title='td.title > a' -p Site=span.sitestr
No      Title   Site
1.      Onedrive is slow on Linux but fast with a ?Windows? user-agent (2016)   microsoft.com
2.      Starting today, users of Firefox can also enjoy Netflix on Linux        netflix.com
3.      Research Debt   distill.pub
...
27.     USPS Informed Delivery ? Digital Images of Front of Mailpieces  usps.com
28.     Performance bugs ? the dark matter of programming bugs  forwardscattering.org
29.     Most items of clothing have complicated international journeys  bbc.co.uk
30.     High-performance employees need quieter work spaces     qz.com
```

It's poor man's scrapper tool if text are the only thing needed. For scrapping beyond text, then just go one step further, to use  [andrew-d/goscrape](https://github.com/andrew-d/goscrape) ( or my [goscrape](https://github.com/suntong/goscrape) instead, which has some enhancements to it).

Again, if text are the only thing needed, then `cascadia` might be already enough. Here is how to scrap Hacker News _across several pages_:

```
$ curl --silent https://news.ycombinator.com/news?p=[1-3] | cascadia -i -o -c 'tr.athing' -p No=span.rank -p Title='td.title > a' -p Site=span.sitestr
No      Title   Site
1.      Starting today, users of Firefox can also enjoy Netflix on Linux        netflix.com
2.      Onedrive is slow on Linux but fast with a ?Windows? user-agent (2016)   microsoft.com 
3.      Research Debt   distill.pub
...
27.     Yes I Still Want to Be Doing This at 56 (2012)  thecodist.com
28.     Performance bugs ? the dark matter of programming bugs  forwardscattering.org
29.     USPS Informed Delivery ? Digital Images of Front of Mailpieces  usps.com
30.     High-performance employees need quieter work spaces     qz.com
31.     Most items of clothing have complicated international journeys  bbc.co.uk
32.     Telstra?s Gigabit Class LTE Network     cellularinsights.com
...
58.     The New Laptop Ban Adds to Travelers' Lack of Privacy and Security      eff.org 
59.     QEMU: user-to-root privesc inside VM via bad translation caching        chromium.org
60.     Startups that debuted at Y Combinator W17 Demo Day 2    techcrunch.com
61.     The Cracking Monolith: Forces That Call for Microservices       semaphoreci.com 
62.     Amsterdam Airport Launches API Platform schiphol.nl
...
88.     Founder Stories: Leah Culver of Breaker (YC W17)        ycombinator.com 
89.     Find out what you, or someone on your team, did on the last working day github.com
90.     PSD2 ? a directive that will change banking in Europe   evry.com
```

By default it uses tab `\t` as fields delimiter, so the output is in `.tsv` format. To change to `.csv`, add `-d ,` to the command line.
