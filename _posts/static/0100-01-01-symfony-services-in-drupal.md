---
layout: book
title: Symfony services in Drupal
permalink: /how/symfony-services-in-drupal
prev_section: /how/integrating-with-symfony-httpkernel
---

It was necessary to hack Drupal core beyond just replacing the bootstrap.

### Headers

Drupal uses PHP's built in functions for HTTP headers, but Symfony's response object should handle headers for us instead. PHP's functions echo their headers immediately, which gives us no opportunity to inspect or change them. It was necessary to hack core to replace all header() function calls with appropriate method calls on a Symfony response object.

I created a Symfony response object as a Symfony service, which I then injected into the PHP $GLOBALS array so that I can access this response object throughout the application.

Headers are set on this response object during bootstrap, and the return value of the Drupal page callback is set as the response content.

### Symfony services in Drupal

I treat the PHP $GLOBALS as a service registry by implementing \ArrayAccess interface around it. This allows me to inject services into the global namespace from my Symfony application using [setter injection](http://symfony.com/doc/current/components/dependency_injection/types.html#setter-injection).

In addition to the Symfony response object, I've also shared the Symfony logger and session service with Drupal.

It may be desirable to pass Symfony's database connection or a caching handler to Drupal this way, too, to replace core functionality. Of course, any Symfony service could be passed to Drupal through the $GLOBALS array as long as there aren't any name collisions.

Any global variable can be specified for Drupal from within Symfony's service configuration by adding additional calls to the [bangpound_drupal.globals service](https://github.com/bangpound/drupal-bundle/blob/master/Resources/config/services.yml).
