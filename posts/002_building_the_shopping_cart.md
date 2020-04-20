# Building the Shopping Cart

In our last post, we used React, Ruby on Rails, and Twitter Bootstrap to create a product gallery. Along the way, we added a button labeled "Add to Cart", which unfortunately was not functional. In this post, we're going to focus on those back-end pieces that will allow a potential customer to store their products in a cart. Once we have said shopping cart functionality available to users, we can shepherd them along to a checkout page, which will be the topic of the next post.

## React on Rails clean up

We still have more to build, but before we do that, we should do a bit of clean up.

### Deleting unnecessary files

When generating the project, some of the installers created extra files that we don't actually need. On the Rails side, we should delete the `HelloWorldController`, the `hello_world` directory under `views` as well as the `hello_world.html.erb` layout.

```sh
$ rm -rf app/views/hello_world
$ rm app/controllers/hello_world_controller.rb
$ rm app/views/layouts/hello_world.html.erb
```

## Adding a shopping cart

From our last post, we had a page that looked like this:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/ckr0zddcgzu998kjowq1.png)

In the next few sections, we'll be creating this interaction by adding the appropriate hooks in the React front-end and Rails back-end.

### Turning the button into an HTML form submit

At the moment, we can click on the "Add to Cart" buttons but they do nothing on their own. Let's change that. Whenever a user wants to add something, we'll submit an HTML form which performs a POST to a Rails route that adds the product to the cart. We'll update the `ProductCard` component to look like this:

```js
/* client/bundles/WebStore/components/ProductCard.jsx */

import React from 'react';

export class ProductCard extends React.Component {
  // This method is new
  renderButton() {
    const csrfToken = document.querySelector('meta[name="csrf-token"]').content;

    return (
      <div className="btn-group">
        <form method="POST" action="/shopping_cart/add">
          <input type="hidden" name="authenticity_token" value={csrfToken}/>
          <input type="hidden" name="shopping_cart_item[product_id]" value={this.props.id}/>
          <input type="hidden" name="shopping_cart_item[quantity]" value="1"/>

          <button type="submit" className="btn btn-sm btn-outline-secondary">
            Add to Cart
          </button>
        </form>
      </div>      
    );
  }

  render() {
    const amount = this.props.price.toLocaleString('en-US', {
      style: 'currency',
      currency: 'USD'
    });

    return (
      <div className="col-md-4">
        <div className="card mb-4 box-shadow">
          <div className="product-image d-flex align-items-center justify-content-center">
            <h2>Image placeholder</h2>
          </div>

          <div className="card-body">
            <p className="card-text">
              <strong>{this.props.name}</strong>
              <br/>
              This is a product description.
              It describes the product and gives the reader a reason to buy it.
            </p>

            <div className="d-flex justify-content-between align-items-center">
              {this.renderButton()}

              <div>{amount}</div>
            </div>
          </div>
        </div>
      </div>
    );
  }
}
```

Take note of the new method `renderButton`, which includes a `<form>` tag. This form provides three parameters in it's payload:

* The CSRF token, which is required to send a POST to the Rails backend.
* The product ID, which tells the backend which product the user is adding to their shopping cart.
* The quantity, which is set to a static default of 1. In the future, the customer could be given the chance to choose more than one.

This payload is enough to add the product, but we do not yet have a shopping cart to send it to. In our form, we specify the URL endpoint as `/shopping_cart/add`, but no such route exists yet in our Rails application. In the next section we will create a new customer object for each guest user session. Attached to that customer object, we can attach a shopping cart.

### Adding custonmers

Whenever a new user comes to our site, we want to track their shopping cart by their cookie. Some e-commerce sites require customers to first create an account. I personally think this is heavy handed, so in our architecture, we are only going to rely on the user's session cookie.

By default, Rails provides us with the `_$APP_session` cookie (where `$APP` is the name of our Rails application). We can get access to this via `session.id` in the controller.

Let us create a `Customer` model that is identified by this session ID. In the future, we may want to convert this `Customer` model into an account, which means letting the user log in to their past orders. Run the following Rails generator:

```sh
$ rails generate model Customer session_id:string
```

Don't forget to migrate!

```sh
$ rake db:migrate
```

Now have a way to reference customers by a session ID. The next step is to automatically look them up each time we need them. This could happen anywhere in our application, so we're adding a helper method to the base controller of all controllers, namely `ApplicationController`. Make the following update to `app/controllers/application_controller.rb`

```rb
# app/controllers/application_controller.rb

class ApplicationController < ActionController::Base
  helper_method :current_customer

  def current_customer
    @current_customer ||= Customer.find_or_create_by(session_id: session.id.to_s)
  end
end
```

The `current_customer` method attempts to find the customer based on this session ID, and creates it if it does not exist. Having a customer is a start, but we also want to associate a shopping cart with this customer, which is covered in the next section.

### Attaching a shopping cart for each customer

To each customer, we assign a shopping cart. A customer may have multiple shopping carts, but in reality, we're only concerned with the latest one. We'll create a new model, the `ShoppingCart`, by running a generator:

```sh
$ rails generate model ShoppingCart customer:references
```

Then migrate:

```sh
$ rake db:migrate
```

In order to add items to a shopping cart, we must also create another model, `ShoppingCartItem`, to keep track of which products are being purchased (via the `product_id`), and how many of them (via the `quantity`). We'll run another generator for that as well. It will include a reference to the original `ShoppingCart`, a reference to the `Product`, and a `quantity` column.

```sh
$ rails generate model ShoppingCartItem shopping_cart:references product:references quantity:integer
```

Then migrate:

```sh
$ rake db:migrate
```

After creating this model, go into `app/models/shopping_cart.rb` and update it to look like the following:

```rb
# app/models/shopping_cart.rb

class ShoppingCart < ApplicationRecord
  belongs_to :customer
  has_many :shopping_cart_items
end
```

And `ShoppingCartItem` should look like this:

```rb
# app/models/shopping_cart_item.rb

class ShoppingCartItem < ApplicationRecord
  belongs_to :shopping_cart
  belongs_to :product
  validates :quantity, presence: true
end
```

For the `Customer` model, we'll have the inverse association to `ShoppingCart`. In other words, we'll add a `has_one :shopping_cart`. To ensure that this association always exists, we create the shopping cart immediately after creating the customer. We can do that via  `after_create :create_shopping_cart!`. We also want to add new items to the shopping cart, so we add a convenience method `add_shopping_cart_item` which takes the shopping cart parameters and adds the shopping cart line item.

```rb
# app/models/customer.rb

class Customer < ApplicationRecord
  has_one :shopping_cart

  after_create :create_shopping_cart!

  def add_shopping_cart_item(params)
    shopping_cart.shopping_cart_items.create!(params)
  end  
end
```

In the next section, we will create the `/shopping_cart/add` endpoint, which will consume the `<form>` payload we added earlier to our `ProductCard` React component.

## Add products to our shopping cart

We have a `Customer`, `ShoppingCart`, and `ShoppingCartItem` available as models for our database. We also have an "Add to Cart" button that will POST to `/shopping_cart/add`. We must connect the two together via a Rails route and a controller. Towards this end, let's add a `ShoppingCartController`:

```sh
$ rails generate controller ShoppingCart add
```

This will create the controller `ShoppingCartController`, to which we will add the following methods:

```rb
# app/controllers/shopping_cart_controller.rb

class ShoppingCartController < ApplicationController
  def add
    current_customer.add_shopping_cart_item(shopping_cart_item_parameters)
    redirect_back(fallback_location: products_path)
  end

  private

  def shopping_cart_item_parameters
    params.require(:shopping_cart_item).permit(:product_id, :quantity)
  end
end
```

Don't forget to update the routes file, since this will be a POST, not a GET:

```rb
# config/routes.rb

Rails.application.routes.draw do
  post 'shopping_cart/add'
  resources :products
  root to: 'products#index'
end
```

In the `add` action, we are doing two things. We use the `add_shopping_cart_item` convenience method to create the shopping cart item. The parameters are selected via the `shopping_cart_item_parameters` method. Using the built-in Rails strong parameters, we expect the payload to have the shape:

```rb
# Only an illustrative example, no need to copy this anywhere
{
  shopping_cart_item: {
    product_id: 123,
    quantity: 1
  }
}
```

After that, we redirect the user back to the original product gallery page, via `redirect_back`. These updates mean that we can now add products to our shopping cart.

### Adding product count to the banner

Now let's update our `SiteHeader` component to link to the checkout page. This means making the "In Cart" indicator at the top right of the page clickable.

```js
/* client/bundles/WebStore/components/SiteHeader.jsx */

import React from 'react';

export class SiteHeader extends React.Component {
  render() {
    return (
      <header>
        <div className="navbar navbar-dark bg-dark box-shadow">
          <a href="/" className="navbar-brand d-flex align-items-center">
            <strong>Web Store</strong>
          </a>

          <a href="/checkout" className="navbar-brand">
            In Cart: {this.props.shopping_cart_item_count}
          </a>
        </div>
      </header>
    );
  }
}
```

The `react_component` method allows us to send props to our component. Let's update our layout page to reflect this new data.

```html
<!-- app/views/layouts/application.html.erb -->

<!DOCTYPE html>
<html>
  <head>
    <title>Web Store</title>
    <%= csrf_meta_tags %>

    <%= stylesheet_link_tag 'application', media: 'all' %>
    <%= javascript_pack_tag 'web-store-bundle' %>
  </head>

  <body>
    <!-- Update this line -->
    <%= react_component("SiteHeader", props: {shopping_cart_item_count: current_customer.shopping_cart_item_count}) %>

    <main role="main">
      <%= yield %>
    </main>
  </body>
</html>
```

Let's add a method to `Customer` for `shopping_cart_item_count`:

```rb
# app/models/customer.rb

class Customer < ApplicationRecord
  has_one :shopping_cart

  after_create :create_shopping_cart!

  def add_shopping_cart_item(params)
    shopping_cart.shopping_cart_items.create!(params)
  end  

  # Add this method to the existing class
  def shopping_cart_item_count
    shopping_cart.shopping_cart_items.count
  end
end
```

With all of that back-end work, a little bit of front-end work, we can finally test it out! Make sure you've got your Rails server running:

```sh
$ rails server
```

Try clicking on the "Add to Cart" button:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/4y564i4v6paby9e3836a.png)

You should then see the "In Cart" count update:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/gbh0o07n94nas94fmm95.png)

Success!

## Conclusion

We started this tutorial with a product gallery. In this post, we described how to make the "Add to Cart" button interactive. This included sending a POST to the appropriate endpoint in Rails. The form included the product ID and the quantity.

At the moment, while we are saving each item to our shopping cart, the list of items in our shopping cart is not yet visible. In the next section we will work on the checkout process, allowing the user to pay for their items. See you then!
