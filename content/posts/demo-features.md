---
title: "Demo: Demo Features"
date: 2023-03-25T12:45:21+08:00
draft: true
author: IsaacTheMan
tags: ['python', 'javascript']
categories: ['demo']
featuredImage: 'images/lock_circuit.png'
math: true
---

# Demo Features

This post aims to show all features this site has to offer.

## Basic Markdown

***bold and italics***

~~**strikethrough and bold**~~

~~*strikethrough and italics*~~

~~***bold, italics and strikethrough***~~

> **Fusion Drive** combines a hard drive with a flash storage (solid-state drive) and presents it as a single logical volume with the space of both drives combined.

* Lorem ipsum dolor sit amet
* Consectetur adipiscing elit
* Integer molestie lorem at massa
* Facilisis in pretium nisl aliquet
* Nulla volutpat aliquam velit
  * Phasellus iaculis neque
  * Purus sodales ultricies
  * Vestibulum laoreet porttitor sem
  * Ac tristique libero volutpat at
* Faucibus porta lacus fringilla vel
* Aenean sit amet erat nunc
* Eget porttitor lorem

1. Lorem ipsum dolor sit amet
2. Consectetur adipiscing elit
3. Integer molestie lorem at massa
4. Facilisis in pretium nisl aliquet
5. Nulla volutpat aliquam velit
6. Faucibus porta lacus fringilla vel
7. Aenean sit amet erat nunc
8. Eget porttitor lorem

- [x] Write the press release
- [ ] Update the website
- [ ] Contact the media

In this example, `<section></section>` should be wrapped as **code**.

```js
grunt.initConfig({
  assemble: {
    options: {
      assets: 'docs/assets',
      data: 'src/data/*.{json,yml}',
      helpers: 'src/custom-helpers.js',
      partials: ['src/partials/**/*.{hbs,md}']
    },
    pages: {
      options: {
        layout: 'default.hbs'
      },
      files: {
        './': ['src/templates/pages/index.hbs']
      }
    }
  }
};
```

| Option | Description |
| ------ | ----------- |
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |

[Upstage](https://github.com/upstage/ "Visit Upstage!")

This is a digital footnote[^1].
This is a footnote with "label"[^label]

[^1]: This is a digital footnote
[^label]: This is a footnote with "label"

![Minion](https://octodex.github.com/images/minion.png)

## Extended Markdown

Plugins or addons to regular markdown.

grinning

$c = \pm\sqrt{a^2 + b^2}$ and \\(f(x)=\int_{-\infty}^{\infty} \hat{f}(\xi) e^{2 \pi i \xi x} d \xi\\)

$$
a = \lim_{i = 0}^N \frac{i}{N}
$$

Sample fraction: [90]/[100]

Font-Awesome :(fas fa-campground fa-fw): Be back soon.

That is so funny! :(far fa-grin-tears):

EMOJISSSS: ⚠️🚳🛐🎦

## Shorcodes

This section is about shortcodes.

{{< style "text-align:right; strong {color:#00b1ff;}" >}}
This is a **right-aligned** paragraph.
{{< /style >}}

{{< figure src="/images/lock_circuit.png" title="Lighthouse (figure)" >}}

{{< gist spf13 7896402 >}}

{{< highlight html >}}
<section id="main">
    <div>
        <h1 id="title">{{ .Title }}</h1>
        {{ range .Pages }}
            {{ .Render "summary"}}
        {{ end }}
    </div>
</section>
{{< /highlight >}}

{{< param description >}}

{{< tweet 917359331535966209 >}}

{{< youtube w7Ft2ymGmfc >}}

{{< image src="/images/lock_circuit.png" caption="Lighthouse (`image`)">}}

{{< admonition type=tip title="This is a tip" open=false >}}
A **tip** banner
{{< /admonition >}}
Or
{{< admonition tip "This is a tip" false >}}
A **tip** banner
{{< /admonition >}}

{{< person url="https://evgenykuznetsov.org" name="Evgeny Kuznetsov" nick="nekr0z" text="author of this shortcode" picture="https://evgenykuznetsov.org/img/avatar.jpg" >}}
