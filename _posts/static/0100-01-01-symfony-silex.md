---
layout: book
title: Symfony, Silex and initial failures
permalink: /how/symfony-and-silex
prev_section: /how/background
next_section: /how/drupal-7-barriers
---

I started developing with Symfony in May 2013, but I found the intense configuration to be almost bureaucratic in its details. I stumbled across [Silex](http://silex.sensiolabs.org), which is a microframework built from Symfony's fundamental components: foundation, kernel, router, and event dispatcher. Silex has barely any configuration overhead, which makes it very easy to play and experiment outside the Symfony framework's heavy configuration requirements. With Silex, you can just start writing PHP.

For Drupal developers who are intimidated or overwhelmed by the Symfony framework, [Silex is an excellent way to become acquainted with the fundamentals](https://igor.io/2013/09/02/how-heavy-is-silex.html) and learn their benefits before getting bogged down in Symfony's bundles and configuration requirements.

It is entirely possible to build complete web applications on Silex, but once I started to approach the limitations of Silex around configuration and code reusability, I went back to Symfony.

## False starts

My first attempts to use Drupal from within a Silex application was [a simple Silex provider that bootstrapped Drupal](http://github.com/bangpound/php-cms-bootstrap-silex-extension) on demand as a kind of Silex service. This allowed me to read Drupal entities into my Silex application, but it really got hung up about sessions. After Silex started its session, Drupal would try to start its own, which would produce annoying PHP warnings.

I considered using [Drush instead of Drupal's bootstrap](http://drush.ws/docs/bootstrap.html), but this was frustrating, too.

I wrote a [Drupal Kernel](https://github.com/bangpound/drupal-kernel) which was functional but too slow and strange. Here I used the HttpKernel and Symfony Console.

The Symfony request was serialized to a small PHP script. This script deserialized the request, overrode the global PHP variables (`$_GET`, `$_POST`, etc.), bootstrapped Drupal and executed the active menu handler (basically Drupal's front controller). All this happened in a separate PHP CGI process which was completely isolated from the original web request. The Drupal response was echoed to stdout, and this output was parsed and turned into a Symfony Response object.

See the [README](https://github.com/bangpound/drupal-kernel/blob/master/README.md).
