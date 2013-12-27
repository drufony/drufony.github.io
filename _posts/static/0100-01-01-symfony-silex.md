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

My first attempts to use Drupal from within a Silex application was a simple Silex provider that bootstrapped Drupal on demand as a kind of Silex service. This allowed me to read Drupal entities into my Silex application, but it really got hung up about sessions. After Silex started its session, Drupal would try to start its own, which would produce annoying PHP warnings.

I considered using Drush instead of Drupal's bootstrap, but this was frustrating, too.
