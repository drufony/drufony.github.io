---
layout: book
title: Drupal 7's barriers to integration
permalink: /how/drupal-7-barriers
prev_section: /how/symfony-and-silex
next_section: /how/drupal-7-similarities
---

If the goal is to run Drupal 7 on top of Symfony, then the place to start picking it apart is index.php.

{% highlight php %}<?php

/**
 * @file
 * The PHP page that serves all page requests on a Drupal installation.
 *
 * The routines here dispatch control to the appropriate handler, which then
 * prints the appropriate page.
 *
 * All Drupal code is released under the GNU General Public License.
 * See COPYRIGHT.txt and LICENSE.txt.
 */

/**
 * Root directory of Drupal installation.
 */
define('DRUPAL_ROOT', getcwd());

require_once DRUPAL_ROOT . '/includes/bootstrap.inc';
drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);
menu_execute_active_handler();
{% endhighlight %}<!-- ?> -->

This file does three things:

* It sets the current working directory to the Drupal root, a small but very important thing that [is better not to fight against](https://drupal.org/node/1928072).
* Bootstraps Drupal, a heavy, monolithic, requisite and unalterable process that is generic to every request.
* Processes the specific request.

## Bootstrap

Drupal 7's bootstrap routines have many overlaps with the Symfony framework. The `drupal_bootstrap()` function acts as a gatekeeper to assure that bootstrap phases run in order and only run once.

To make the bootstrap overrideable, I replaced the various bootstrap functions with a [Pimple](http://pimple.sensiolabs.org) container. Pimple is a very lightweight dependency injection service container which is used in Silex instead of the more complex DependencyInjection container component of Symfony.

For my purposes, it works well because it is a PHP class that can be overridden, and services are defined as closures or anonymous functions. When a service is requested from Pimple, the function that produces the service is executed once and cannot be executed again.

In my Pimple-based bootstrap, the PHP magic method `__invoke()` performs the same duties as `drupal_bootstrap()`. Each bootstrap phase is one or more services. The `drupal_bootstrap()` function then manages a single instance of the Bootstrap Pimple object, and it can be called from anywhere in Drupal without any changes. `drupal_bootstrap()` function signature has changed to allow a different Pimple bootstrap container to be specified.

Replacing the bootstrap with a Pimple container required hacking core, but it remains completely compatible with everything else in Drupal. Drupal's tests still pass.

* [Differences between Drupal 7 and Drupal 7 Pimple](https://github.com/bangpound/drupal/compare/7.x...7.x-pimple)

## Autoloading

Now that the bootstrap can be overridden, I can substitute different functionality for Drupal's original bootstrap phases. One of the most annoying features of Drupal 7 has been the class registry which requires the database. (If you have had to use `drush registry-rebuild` you know how the registry easily breaks especially when you're moving your Drupal database between environments.)

We can use composer's autoloading instead. While Drupal 7's registry only scans enabled modules for classes, it's legitimate to include all classes in the autoloader. However, Drupal 7's profiles and multisite architecture require some special handling.

Each location where modules can be installed now needs a composer.json file even if there are no third-party libraries. Composer can generate a classmap for a project, and if the autoload.php file for each of these locations is loaded in the correct order (Drupal root, profile, sites/all, and sites/*), module specificity is maintained.

By replacing Drupal 7's registry with PSR-0 autoloading provided by composer, we can remove a significant part of Drupal's bootstrap and make it composer's responsibility. This also begins to disentangle the bootstrap process.

* [Differences between Drupal 7 and Drupal 7 Autoload](https://github.com/bangpound/drupal/compare/7.x...7.x-autoload)
* [Differences between Drupal 7 Pimple and Drupal 7 Autoload](https://github.com/bangpound/drupal/compare/7.x-pimple...7.x-autoload)

## Other bootstrap hurdles and opportunities

Looking at `drupal_bootstrap()` from the perspective of my Symfony experience, it's clear that what Drupal's bootstrap is doing is service configuration and instantiation. The services it creates are:

* Session handler
* Database connections
* Cache handler
* Lock handler
* Site configuration variables
* Exception and error handling and logging
* Autoloading

In addition to setting up these services, `drupal_bootstrap()` also:

* Optimizes PHP ini settings for Drupal, including its session handler.
* Stops malicious requests.
* Redirects to install.php if the site settings do not exist or the database is empty.
* Delivers a cached page.
* Sets HTTP headers.
* Delivers 404 responses.
* Starts the session.
* Loads the logged in user.

These responsibilities are conflated in `drupal_bootstrap()` possibly for performance. Bootstrapping is a big task, and if a page is cached or the site is unavailable, determining that early and completing the request sooner saves resources. However, for Symfony, these tasks need to be separated or disabled completely. So for Symfony's Drupal bootstrap, I do not support cached pages (Symfony offers a fine file-based cache for page responses). I also do not support installation from bootstrap. I let Symfony handle exceptions and errors.

Although maintenance themes and "site offline" functionality is not part of bootstrap, it's another significant feature that I dropped to reduce complexity.
