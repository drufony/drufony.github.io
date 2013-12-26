---
layout: default
title: How it is done
---

# Porting Drupal 7 to the Symfony 2 Framework

I have spent the last few months learning the Symfony framework and building a few web applications with it, and my interest was provoked by the Drupal community's decision to use Symfony components in Drupal 8. I have been quite pleased by the relative ease in picking up the Symfony framework, but it's hard to leave Drupal 7 behind, and I can't wait for Drupal 8 to stabilize. So I decided to try porting Drupal 7 to Symfony.

## Symfony and Silex

I started developing with Symfony in May 2013, but I found the intense configuration to be almost bureaucratic in its details. I stumbled across [Silex](http://silex.sensiolabs.org), which is a microframework built from Symfony's fundamental components: foundation, kernel, router, and event dispatcher. Silex has barely any configuration overhead, which makes it very easy to play and experiment outside the Symfony framework's heavy configuration requirements. With Silex, you can just start writing PHP.

For Drupal developers who are intimidated by the Symfony framework, [Silex is an excellent way to become acquainted with the fundamentals](https://igor.io/2013/09/02/how-heavy-is-silex.html) and learn their benefits before getting bogged down in Symfony's bundles and configuration requirements.

It is entirely possible to build complete web applications on Silex, but once I started to approach the limitations of Silex around configuration and code reusability, I went back to Symfony.

## False starts

My first attempts to use Drupal from within a Silex application was a simple Silex provider that bootstrapped Drupal on demand as a kind of Silex service. This allowed me to read Drupal entities into my Silex application, but it really got hung up about sessions. After Silex started its session, Drupal would try to start its own, which would produce annoying PHP warnings.

I considered using Drush instead of Drupal's bootstrap, but this was frustrating, too.

## Drupal 7's barriers to integration

If the goal is to run Drupal 7 on top of Symfony, then the place to start picking it apart is index.php.

    define('DRUPAL_ROOT', getcwd());

    require_once DRUPAL_ROOT . '/includes/bootstrap.inc';
    drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);
    menu_execute_active_handler();

Here we see two hurdles:

* Drupal requires that the Drupal root also be PHP's current working directory.
* Drupal's bootstrap process is monolithic and unalterable.

## Bootstrap

Drupal 7's bootstrap routines have many overlaps with the Symfony framework. The drupal_bootstrap() function acts as a gatekeeper to assure that bootstrap phases run in order and only run once.

To make the bootstrap overrideable, I replaced the various bootstrap functions with a Pimple container. Pimple is a very lightweight service container (or service locator depending on who you ask) which is used in Silex instead of the more complex DependencyInjection container component of Symfony. For my purposes, it works well because it is a class that can be overridden, and services are defined as closures or anonymous functions. When a service is requested from Pimple, the function that produces the service is executed once and cannot be executed again.

In my Pimple-based bootstrap, the PHP magic method \_\_invoke() performs the same duties as drupal\_bootstrap(). Each bootstrap phase is one or more services. The drupal_bootstrap() function then manages a single instance of the Bootstrap Pimple object, and it can be called from anywhere in Drupal without any changes. drupal_bootstrap() function signature has changed to allow a different Pimple bootstrap container to be specified.

Replacing the bootstrap with a Pimple container required hacking core, but it remains completely compatible with everything else in Drupal. Drupal's tests still pass.

## Autoloading

Once the bootstrap can be overridden, we can begin substituting different functionality for Drupal's original bootstrap phases. One of the most annoying features of Drupal 7 has been the class registry which requires the database. (If you have had to use `drush registry-rebuild` you know how the registry easily breaks especially when you're moving your Drupal database between environments.)

These days, we can use composer's autoloading instead. While Drupal 7's registry only scans enabled modules for classes, it's legitimate to include all classes in the autoloader. However, Drupal 7's profiles and multisite architecture require some special handling.

Each location where modules can be installed now needs a composer.json file even if there are no third-party libraries. Composer can generate a classmap for a project, and if the autoload.php file for each of these locations is loaded in the correct order (Drupal root, profile, sites/all, and sites/*), module specificity is maintained.

By replacing Drupal 7's registry with PSR-0 autoloading provided by composer, we can remove a significant part of Drupal's bootstrap and make it composer's responsibility. This also disentangles the bootstrap process.

## Other bootstrap hurdles

Looking at drupal_bootstrap() from the perspective of my Symfony experience, it's clear that what Drupal's bootstrap is doing is service configuration and instantiation. The services it creates are:

* Session handler
* Database connections
* Cache handler
* Lock handler
* Site configuration variables
* Exception and error handling and logging
* Autoloading

In addition to setting up these services, drupal_bootstrap() also:

* Optimizes PHP ini settings for Drupal, including its session handler.
* Stops malicious requests.
* Redirects to install.php if the site settings do not exist or the database is empty.
* Delivers a cached page.
* Sets HTTP headers.
* Delivers 404 responses.
* Starts the session.
* Loads the logged in user.

These responsibilities are conflated in drupal_bootstrap() mostly for performance. Bootstrapping is a big task, and if a page is cached or the site is unavailable, determining that early and completing the request sooner saves resources. However, for Symfony, these tasks need to be separated or disabled completely. So for Symfony's Drupal bootstrap, we do not support cached pages (Symfony offers a fine file-based cache for page responses). We also do not support installation from bootstrap. We let Symfony handle exceptions and errors.

Although maintenance themes and "site offline" functionality is not part of bootstrap, it's another significant feature that I dropped to reduce complexity.

## The Symfony kernel

In Symfony, the HttpKernelInterface is a PHP interface that models one basic task: converting a web request into a web response. The HttpKernel class which implements this interface triggers a few events that allow you to change or interrupt a request or response as it moves through this process.

Understanding the HttpKernel is crucial, and I won’t try to repeat the [existing documentation](http://symfony.com/doc/current/components/http_kernel/introduction.html#http-kernel-working-example), but here are some pointers.

* [Value of HttpFoundation](https://igor.io/2013/02/03/http-foundation-value.html) by Igor Wiedler. HttpFoundation is the Symfony component that contains the building blocks of requests, responses, sessions, cookies, etc. HttpKernel depends on it.
* [Symfony2 components overview: HttpKernel](http://blog.servergrove.com/2013/09/30/symfony2-components-overview-httpkernel/) by Raul Fraile. This is probably the briefest explanation of the HttpKernel available, and [all of Fraile’s posts on the topic](http://blog.servergrove.com/tag/symfony2-components/) are worth reading.
* [Create your own framework... on top of the Symfony2 Components](http://fabien.potencier.org/article/50/create-your-own-framework-on-top-of-the-symfony2-components-part-1) by Fabien Potencier. This series of twelve blog posts is a tutorial that is worth reading in its entirety, but the HttpKernel is introduced in part 6. Events are covered in part 9 and 11. The whole tutorial is important, though!
* Symfony [Internals](http://symfony.com/doc/current/book/internals.html) documentation covers HttpFoundation, HttpKernel, events and the Symfony framework bundle which ties all of the components together.

In addition to this documentation, studying how the HttpKernel was used in different contexts was helpful: Silex, [Stack](https://igor.io/2013/02/02/http-kernel-middlewares.html), [Bolt](http://www.bolt.cm), and [Drupal 8](https://drupal.org/drupal-8.0). [YOLO](http://yolophp.com), “an academic version of Silex,” replaces Pimple with the Symfony dependency injection container to create a “more pure, less usable” tool.

## Drupal on HttpKernel

Once drupal_bootstrap() is simplified and overridden, the remaining problem to solve is actually handling the request and returning a response.

At first glance, it would appear that Drupal 7 would require extensive rewrites to even begin to operate within the constraints of the Symfony HttpKernel. Nothing in Drupal 7 uses a Symfony request object, and no page callback returns a Symfony response object.

In Drupal, we have `menu\_execute\_active\_handler()` which reads the current URL path, searches for a route that matches that path, calls the page callback, and renders the result.

    if ($router_item = menu_get_item($path)) {
      if ($router_item['access']) {
        if ($router_item['include_file']) {
          require_once DRUPAL_ROOT . '/' . $router_item['include_file'];
        }
        $page_callback_result = call_user_func_array($router_item['page_callback'], $router_item['page_arguments']);
      }
      else {
        $page_callback_result = MENU_ACCESS_DENIED;
      }
    }
    else {
      $page_callback_result = MENU_NOT_FOUND;
    }

menu_execute_active_handler() roughly maps to the Symfony HttpKernel's handle() method. In fact, they both have a significant element in common.

In Drupal's menu_execute_active_handler()

    $page_callback_result = call_user_func_array($router_item['page_callback'], $router_item['page_arguments']);

looks remarkably like HttpKernel::handle().

    $response = call_user_func_array($controller, $arguments);

Because the HttpKernel triggers events at many different points in this function, we can coax Drupal’s page callbacks to work like Symfony controllers.

### Routing

Both Drupal and Symfony have a router which defines paths, controllers, default arguments, and validity requirements. It is simple to achieve a very basic representation of Drupal’s menu_router table as a Symfony route collection. In the DrupalLoader class, each Drupal route (such as node/%/edit) is rewritten to use Symfony’s route patterns (becomes node/{p0}/edit).

All of the data attached to the route objects in the loader can be retrieved from the request object when that route is matched. My custom Drupal route loader only registers these routes and flags them as Drupal routes. Drupal has two representations of routes: a generic one that can be loaded from the menu_router table and one retrieved from menu_get_item() that is specific for the user and context. The latter is more complete, but it’s also going to be different from user to user. Symfony wants to cache its routes, so we have to use the generic route.

### kernel.request

The first event dispatched by the kernel is [kernel.request](http://symfony.com/doc/current/components/http_kernel/introduction.html#the-kernel-request-event). Event listeners can change the request or return a response.

Thomas Rabaix of Ekino [released a Symfony bundle](http://www.ekino.com/drupal-and-symfony2-dont-wait-for-drupal8/) that integrates with Drupal entirely through [the request event](https://github.com/ekino/EkinoDrupalBundle/blob/master/Drupal/DrupalRequestListener.php). This effectively short circuits the HttpKernel, and it does not attempt to call a controller.

While this works and is functional, it means that Drupal page callbacks are not really fully integrated into the HttpKernel. Instead, I use the request event to modify the request by adding a _controller attribute that specifies the Drupal page callback. The request listener also includes the file that contains the page callback and throws an exception when the user doesn’t have permission to access that page.

The request listener also attaches the controller arguments to the request.

### Controller and arguments

After dispatching the kernel.request event, the HttpKernel wants to call the controller. It acquires the arguments (which were stashed away in the request object earlier) from the ControllerResolver.

The HttpKernel resembles Drupal menu_execute_active_handler(), but we’ve only determined the controller. It acquires the arguments from the ControllerResolver, which I’ve overridden to simply return the arguments attached to the Drupal-flagged requests by the request listener.

### Turning return values into response objects

Most Drupal page callbacks in core will return a string or an array. Converting these into Symfony response objects is actually very simple, because HttpKernel dispatches the kernel.view event when the controller returns something that is *not* a Symfony response object.

The ViewListener picks up the controller’s return value and performs the rest of the work Drupal does in `menu_execute_active_handler()`. It calls `drupal_deliver_page()` or another Drupal delivery callback, which ultimately results in a string. This string is the Symfony response content.

### Turning NULL into response objects

In many cases, Drupal’s `menu_execute_active_handler()` never finishes because the page callback never returns anything. Sometimes, the page callback deliberately calls drupal_exit() after echoing a string. drupal_goto(), ajax handlers, autocomplete field callbacks, and image generators also behave this way.

This kind of behavior is not supported by Symfony HttpKernel, but it’s not hard to implement. We need the Symfony HttpKernel to recover from a PHP shutdown.

I wrote a new ShutdownableInterface for HttpKernel which defines a new method that finishes handling the request and response. When PHP shutdown happens during HttpKernel’s controller phase, an the contents of the PHP output buffer are converted to a Symfony response, which is passed to HttpKernel::shutdown().

### Headers

Drupal uses PHP’s built in functions for HTTP headers, but Symfony’s response object should handle headers for us instead. PHP’s functions echo their headers immediately, which gives us no opportunity to inspect or change them. It was necessary to hack core to replace all header() function calls with appropriate method calls on a Symfony response object.

I created a Symfony response object as a Symfony service, which I then injected into the PHP $GLOBALS array so that I can access this response object throughout the application.

Headers are set on this response object during bootstrap, and the return value of the Drupal page callback is set as the response content.

### Symfony services in Drupal

I treat the PHP $GLOBALS as a service registry by implementing \ArrayAccess interface around it. This allows me to inject services into the global namespace from my Symfony application using [setter injection](http://symfony.com/doc/current/components/dependency_injection/types.html#setter-injection).

In addition to the Symfony response object, I’ve also shared the Symfony logger and session service with Drupal.

It may be desirable to pass Symfony’s database connection or a caching handler to Drupal this way, too, to replace core functionality. Of course, any Symfony service could be passed to Drupal through the $GLOBALS array as long as there aren’t any name collisions.

Any global variable can be specified for Drupal from within Symfony this way.
