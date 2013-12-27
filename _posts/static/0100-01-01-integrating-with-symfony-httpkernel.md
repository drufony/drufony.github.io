---
layout: book
title: Integrating with Symfony Kernel and HttpKernel
permalink: /how/integrating-with-symfony-httpkernel
prev_section: /how/routing
next_section: /how/symfony-services-in-drupal
---

The Symfony Kernel (or AppKernel) is the core of a Symfony application which organizes bundles and handles Symfony's bootstrapping routines. During a request, the Kernel is instantiated before the HttpKernel.

To support Drupal, I have extended the HttpKernel with a shutdown handler. The AppKernel instantiates the [Drupal bundle, which takes care of bootstrapping Drupal](https://github.com/bangpound/drupal-bundle/blob/master/BangpoundDrupalBundle.php).

## Bootstrap

The bundle's `boot()` method calls `drupal_bootstrap()` after setting some global variables required for Drupal. It also changes the working directory, which can be problematic for Symfony. The current working directory is only changed if there is actually a web request to process. If the bundle is being booted simply because the Symfony cache is being cleared or Composer dependencies are being updated, the current working directory is not changed.

## HttpKernel request event

The first event dispatched by the kernel is [kernel.request](http://symfony.com/doc/current/components/http_kernel/introduction.html#the-kernel-request-event). Event listeners can change the request or return a response.

Thomas Rabaix of Ekino [released a Symfony bundle](http://www.ekino.com/drupal-and-symfony2-dont-wait-for-drupal8/) that integrates with Drupal entirely through [the request event](https://github.com/ekino/EkinoDrupalBundle/blob/master/Drupal/DrupalRequestListener.php). This effectively short circuits the HttpKernel, and it does not attempt to call a controller.

While this works and is functional, it means that Drupal page callbacks are not integrated into the HttpKernel. Instead, [I use the request event to modify the request](https://github.com/bangpound/drupal-bundle/blob/master/EventListener/RequestListener.php) by setting the `_controller` request attribute that specifies the Drupal page callback. The request listener also includes the file that contains the page callback and throws an exception when the user doesn't have permission to access that page.

The request listener also attaches the controller arguments to the request.

## Controller and arguments

After dispatching the kernel.request event, the HttpKernel wants to call the controller. It acquires the arguments (which were stashed away in the request object earlier) from the ControllerResolver.

The HttpKernel resembles Drupal `menu_execute_active_handler()`, but so far, I have only determined the name of the controller function. HttpKernel acquires the arguments from the [ControllerResolver](https://github.com/bangpound/drupal-bundle/blob/master/Controller/ControllerResolver.php), which I've overridden to simply return the arguments attached to the Drupal-flagged requests by the request listener.

## Turning return values into response objects

Most Drupal page callbacks in core will return a string or an array. Converting these into Symfony response objects is actually very simple, because HttpKernel dispatches the `kernel.view` event when the controller returns something that is *not* a Symfony response object.

The [ViewListener](https://github.com/bangpound/drupal-bundle/blob/master/EventListener/ViewListener.php) picks up the controller's return value and performs the rest of the work Drupal does in `menu_execute_active_handler()`. It calls `drupal_deliver_page()` or another Drupal delivery callback, which ultimately results in a string. This string is the Symfony response content.

## Turning NULL into response objects

In many cases, Drupal's `menu_execute_active_handler()` never finishes because the page callback never returns anything. Sometimes, the page callback deliberately calls `drupal_exit()` after echoing a string. `drupal_goto()`, ajax handlers, autocomplete field callbacks, and image generators also behave this way.

This kind of behavior is not supported by Symfony HttpKernel, but it's not hard to implement. I just need the Symfony HttpKernel to recover from (or finish up after) a PHP shutdown.

I wrote a new [ShutdownableInterface](https://github.com/bangpound/drupal-bundle/blob/master/HttpKernel/ShutdownableInterface.php) for HttpKernel which defines a new method that finishes handling the request and response. When PHP shutdown happens during HttpKernel's controller phase, and the contents of the PHP output buffer are converted to a Symfony response, which is passed to HttpKernel::shutdown().

The [ShutdownListener](https://github.com/bangpound/drupal-bundle/blob/master/EventListener/ShutdownListener.php) listens on almost every kernel event to first set up the PHP shutdown handler and then to disable or ignore it if the controller returned a value.
