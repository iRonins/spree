---
title: Logic Customization
section: customization
---

This guide explains how to customize the internal Spree code to meet
your exact business requirements including:

-   Extending and overriding existing Spree models and controllers
-   Changing the output from an existing Spree controller action
-   Customizing the image handling functionality.

### Extending Classes

All of Spree's business logic (models, controllers, helpers, etc) can
easily be extended / overridden to meet your exact requirements using
standard Ruby idioms.

**Using Ruby 2.0+ fight the war on class_eval.**

<% ruby do %>
    module ProductExtensions
      extend ActiveSupport::Concern

      included do
        singleton_class.prepend ClassMethods
        prepend InstanceMethods
      end

      module ClassMethods
        def some_class_method
           ...
        end
      end

      module InstanceMethods
        def instance_method_override
           # do before work
           original_result = super # do original work
           # do after work
           return original_result
         end
         def new_instance_method
           # do something new
         end
      end
    end
    Spree::Product.include ProductExtensions
<% end %>

**Using Ruby 1.9.3.**

Standard practice for including such changes in your application or
extension is to create a file within the relevant **app/models/spree** or
**app/controllers/spree** directory with the original class name with
**_decorator** appended.

**Adding a custom method to the Product model:**
app/models/spree/product_decorator.rb

<% ruby do %>
    Spree::Product.class_eval do
      def some_method
        ...
      end
    end
<% end %>

**Adding a custom action to the ProductsController:**
app/controllers/spree/products_controller_decorator.rb

<% ruby do %>
    Spree::ProductsController.class_eval do
      def some_action
        ...
      end
    end
<% end %>

***
The following code is a preferred way when redefining existing methods,
because it allows you to invoke original behaviour by calling original
method with "super".
***

**Redefining a method to the Product model:**
app/models/spree/product_decorator.rb

<% ruby do %>
    module ProductExtensions
      def self.prepended(base)
        base.extend(ClassMethods)
      end
      def instance_method_override
        # do before work
        original_result = super # do original work
        # do after work
        return original_result
      end
      def new_instance_method
        # do something new
      end
      module ClassMethods
        def some_class_method
          ...
        end
      end
    end

    Spree::Product.class_eval do
      prepend ProductExtensions
    end
<% end %>

**Adding behaviour to the ProductsController update action:**
app/controllers/spree/products_controller_decorator.rb

<% ruby do %>
    module ProductsControllerExtensions
      def update
        do_before_work
        super
        do_after_work
      end

      protected
      def do_before_work
        ...
      end
      def do_after_work
        ...
      end
    end

    Spree::ProductsController.class_eval do
      prepend ProductsControllerExtensions
    end
<% end %>

#### Accessing Product Data

If you extend the Products controller with a new method, you may very
well want to access product data in that method. You can do so by using
the :load_data before_filter.

<% ruby do %>
    Spree::ProductsController.class_eval do
      before_filter :load_data, :only => :some_action

      def some_action
        ...
      end
    end
<% end %>

***
:load_data will use params[:id] to lookup the product by its
permalink.
***

### Overriding Controller Action Responses

With the release of 0.60.0 Spree now supports a new way of overriding or
changing the output of an existing controller's action without needing
to completely override the method (while also easily avoiding double
render exceptions).

#### respond_override method

The **respond_override** method is used to customize the response from
any action, and is built on top of Rails 3's **respond_with** method
(that all Spree controllers are now using). It accepts a hash of options
using the following syntax:

<% ruby do %>
    respond_override :action_name => { :format =>  { :result => lambda { ... response ... } } }
<% end %>

-   **:action_name** - Can be any existing action within a controller
    (i.e. :update, :create, :new), provided that action is using
    respond_with to define its response.
-   **:format** - The format of the request, (i.e. :html, :json, :js,
    etc). All Spree controllers have a class level **respond_to**
    method call that defines which formats the controller's actions will
    respond to.
-   **:result** - Two possible results are available on any given
    response, :success or :failure. :success being the default for most
    actions. However, actions that change or create models will have a
    :failure response if validation fails for the model being updated.
-   **lambda** - The lambda passed contains the actual code to create
    the desired custom response (i.e. render or redirect_to). A lambda
    must be passed to ensure the code is evaluated at the correct time.

#### Example Usage

If you wanted to render a custom partial for the index action of
ProductsController, you could include the following in your
**app/controllers/spree/products_controller_decorator.rb** file.

<% ruby do %>
    Spree::ProductsController.class_eval do
      respond_override :index => { :html =>
        { :success => lambda { render 'shared/some_file' } } }
    end
<% end %>

Or if you wanted to redirect on the failure to create in
Admin::ProductsController, you would use:

<% ruby do %>
    Spree::Admin::ProductsController.class_eval do
      respond_override :create => { :html => { :failure => lambda {
        redirect_to some_url } } }
    end
<% end %>

#### Caveats

-   If an action does not use **respond_with** to define its response
    the **respond_override** will not work.
-   Some actions contain several **respond_with** calls so any
    **respond_override** defined on it will be executed for any of the
    **respond_with** instances, so it's important to check the model
    state / logic within the lambda passed to prevent overriding all
    possible responses with the same override.

### Product Images

Spree uses Thoughtbot's
[paperclip](https://github.com/thoughtbot/paperclip) gem to manage
images for products. All the normal paperclip options are available on
the Image class. If you want to modify the default Spree product and
thumbnail image sizes, simply create an image_decorator.rb file in your
app model directory, and override the attachment sizes:

<% ruby do %>
    Spree::Image.class_eval do
      attachment_definitions[:attachment][:styles] = {
        :mini => '48x48>', # thumbs under image
        :small => '100x100>', # images on category view
        :product => '240x240>', # full product image
        :large => '600x600>' # light box image
      }
    end
<% end %>

You may also add additional image sizes for use in your templates
(:micro for shopping cart view, for example).

#### Image resizing option syntax

Default behavior is to resize the image and maintain aspect ratio (i.e.
the :product version of a 480x400 image will be 240x200). Some commonly
used options are:

-   trailing #, image will be centrally cropped, ensuring the requested
dimensions
-   trailing >, image will only be modified if it is currently larger
than the requested dimensions. (i.e. the :small thumb for a 100x100
original image will be unchanged)
