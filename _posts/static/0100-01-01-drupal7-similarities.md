---
layout: book
title: Drupal 7's similarities
permalink: /how/drupal-7-similarities
prev_section: /how/drupal-7-barriers
next_section: /how/routing
---

Once `drupal_bootstrap()` is simplified and overridden, the remaining problem to solve is actually handling the request and returning a response.

You might assume that Drupal 7 would require extensive rewrites to even begin to operate within the constraints of the Symfony HttpKernel. Nothing in Drupal 7 uses a Symfony request object, and no page callback returns a Symfony response object.

In Drupal, we have [`menu_execute_active_handler()`](https://api.drupal.org/api/drupal/includes%21menu.inc/function/menu_execute_active_handler/7) which reads the current URL path, searches for a route that matches that path, calls the page callback, and renders the result.

{% highlight php %}<?php
function menu_execute_active_handler($path = NULL, $deliver = TRUE) {
  // Check if site is offline.
  $page_callback_result = _menu_site_is_offline() ? MENU_SITE_OFFLINE : MENU_SITE_ONLINE;

  // Allow other modules to change the site status but not the path because that
  // would not change the global variable. hook_url_inbound_alter() can be used
  // to change the path. Code later will not use the $read_only_path variable.
  $read_only_path = !empty($path) ? $path : $_GET['q'];
  drupal_alter('menu_site_status', $page_callback_result, $read_only_path);

  // Only continue if the site status is not set.
  if ($page_callback_result == MENU_SITE_ONLINE) {
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
  }

  // Deliver the result of the page callback to the browser, or if requested,
  // return it raw, so calling code can do more processing.
  if ($deliver) {
    $default_delivery_callback = (isset($router_item) && $router_item) ? $router_item['delivery_callback'] : NULL;
    drupal_deliver_page($page_callback_result, $default_delivery_callback);
  }
  else {
    return $page_callback_result;
  }
}
{% endhighlight %}<!-- ?> -->

I'll break this function into three parts:

* The beginning with the comment `Check if site is offline` is something I abandoned entirely. If the site is offline, Symfony or the web server will handle it. ([Stack Backstage](https://github.com/atst/stack-backstage) is an HttpKernel-based alternative that provides similar functionality.)
* The middle which begins with the comment `Only continue if the site status is not set` calls the Drupal page callback.
* The end which begins with the comment `Deliver the result of the page callback to the browser` sends the page callback result to another function called a "delivery callback" which will actually render the page or *something* to the user.

## `menu_execute_active_handler()` is `HttpKernelInterface::handle()`

`menu_execute_active_handler()` roughly maps to the Symfony `HttpKernelInterface::handle()` method. In fact, they both have a significant element in common.

In Drupal's `menu_execute_active_handler()`:

{% highlight php %}<?php
$page_callback_result = call_user_func_array($router_item['page_callback'], $router_item['page_arguments']);
{% endhighlight %}<!-- ?> -->

looks remarkably like `HttpKernel::handle()`:

{% highlight php %}<?php
$response = call_user_func_array($controller, $arguments);
{% endhighlight %}<!-- ?> -->

Because the HttpKernel triggers events at many different points in this function, we can coax Drupal's page callbacks to work like Symfony controllers.
