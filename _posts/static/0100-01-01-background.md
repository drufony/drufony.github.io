---
title: Background reading
layout: book
permalink: /how/background
prev_section: /how
next_section: /how/symfony-and-silex
---

<p class="lead">To integrate Drupal with Symfony, I had to let Symfony be the boss of Drupal, which required a full understanding of Symfony's HttpKernel component.</p>

In Symfony, HttpKernelInterface is a PHP interface that models one basic task: converting a web request into a web response. The HttpKernel class which implements this interface triggers a few events that allow you to change or interrupt a request or response as it moves through this process.

Understanding the HttpKernel is crucial, and I won't try to repeat the [existing documentation](http://symfony.com/doc/current/components/http_kernel/introduction.html#http-kernel-working-example), but here are some pointers that were very helpful to me.

* [Value of HttpFoundation](https://igor.io/2013/02/03/http-foundation-value.html) by Igor Wiedler. HttpFoundation is the Symfony component that contains the building blocks of requests, responses, sessions, cookies, etc. HttpKernel depends on it.
* [Symfony2 components overview: HttpKernel](http://blog.servergrove.com/2013/09/30/symfony2-components-overview-httpkernel/) by Raul Fraile. This is probably the briefest explanation of the HttpKernel available, and [all of Fraile's posts on the topic](http://blog.servergrove.com/tag/symfony2-components/) are worth reading.
* [Create your own framework... on top of the Symfony2 Components](http://fabien.potencier.org/article/50/create-your-own-framework-on-top-of-the-symfony2-components-part-1) by Fabien Potencier. This series of twelve blog posts is a tutorial that is worth reading in its entirety, but the HttpKernel is introduced in part 6. Events are covered in part 9 and 11. The whole tutorial is important, though!
* Symfony [Internals](http://symfony.com/doc/current/book/internals.html) documentation covers HttpFoundation, HttpKernel, events and the Symfony framework bundle which ties all of the components together.

In addition to this documentation, studying how the HttpKernel was used in different contexts was helpful: [Silex](http://silex.sensiolabs.org), [Stack](https://igor.io/2013/02/02/http-kernel-middlewares.html), [Bolt](http://www.bolt.cm), and [Drupal 8](https://drupal.org/drupal-8.0). [YOLO](http://yolophp.com), "an academic version of Silex," replaces Pimple with the Symfony dependency injection container to create a "more pure, less usable" tool.
