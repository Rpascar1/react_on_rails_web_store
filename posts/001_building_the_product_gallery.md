# React on Rails Web Store

Robust e-Commerce web applications are very complex, and creating one can be an amazing learning experience. Towards this end, we are going to go through all of the steps of creating one from scratch using Ruby on Rails on the backend and React on the front-end. Because the process can be long, we'll be breaking this up into multiple posts. In this post we will focus on creating a static product gallery. Making a shopping cart will come in the next post.

## Creating a Rails project

First create a new Rails project (make sure you are on Rails 5.2 or newer)

```sh
$ rails new web_store --webpack=react
```

The `--webpack` parameter will start us off with a nice `package.json`, but that alone is not necessarily robust for now needs. We're going to use the `react_on_rails` gem as well. Add the following line to your `Gemfile`:

```rb
gem 'react_on_rails', '11.2.2'
```

Make sure to do a `bundle install` after this step.

```sh
$ bundle install
```

The `react_on_rails` gem requires we run a generator. However, before you can run the generator, make sure to commit all of your code.

```sh
$ git add --all
$ git commit -m "Adding react_on_rails"
```

If you don't commit, you'll get an error message:

```shell
$ rails generate react_on_rails:install
Running via Spring preloader in process 83522
ERROR: You have uncommitted code. Please commit or stash your changes before continuing
ERROR: react_on_rails generator prerequisites not met!
```

Got everything committed? Good! Now we can run the `react_on_rails` generator:

```shell
$ rails generate react_on_rails:install
```

This will create a lot of new code under `app/javascript`. While we could keep our JavaScript files here, I prefer to move it to the root level. To make `client` the location of webpacker project, we need to move the files.

```sh
$ mv app/javascript client
```

And we also have to change the `source_path` in `config/webpacker.yml`

```yaml
# config/webpacker.yml
default: &default
  source_path: client # Update this line
  source_entry_path: packs
```

Let's start the project! Run the following `foreman` command (if you don't have foreman, it's just `gem install foreman`):

```sh
$ foreman start -f Procfile.dev
20:55:01 web.1    | started with pid 96362
20:55:01 client.1 | started with pid 96363
20:55:04 web.1    | => Booting Puma
20:55:04 web.1    | => Rails 5.2.4.1 application starting in development
20:55:04 web.1    | => Run `rails server -h` for more startup options
```

You should be able to view the default "Hello World" component at [http://localhost:3000/hello_world](http://localhost:3000/hello_world)

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/5guuc21anv3jnham6kps.png)

Once you've verified that React on Rails is running, let's add the next major gem: Twitter Bootstrap.

### Adding the Twitter Bootstrap gem

A good webstore needs to be designed well, so in that regard, let's create a good starting point for our HTML and CSS by including the Twitter Bootstrap gem to our project. Place this line in your `Gemfile`:

```rb
gem 'bootstrap', '~> 4.4.1'
```

And then install the gem:

```sh
$ bundle install
```

This should give us access to the Twitter Bootstrap SCSS files. But it will not show up in our designs until we do the necessary `@import`. Check if you have a file called `app/assets/stylesheets/application.scss`. If not, you may have to rename `application.css` to `application.scss`:

```sh
$ mv app/assets/stylesheets/application.css app/assets/stylesheets/application.scss
```

Once you have such a file, add this line to `app/assets/stylesheets/application.scss`:

```css
/* app/assets/stylesheets/application.scss */
@import "bootstrap";
```

With that set up, we can build a product gallery where users can begin adding things to their cart.

## Adding a product gallery

Using Twitter Bootstrap as a base, we can create a product gallery. On the Bootstrap examples page, they have an album template:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/kihzo28sprjmlqu6rtuf.png)

We're going to rework this example to become a product gallery.

### Adding RESTful routes for Products

In order for our React client side app to interact with our Rails back-end app, we'll want RESTful routes to be able to get product information. While we might not be using all of these up front, they will be useful in the future.

```sh
$ rails generate scaffold Product name:string image_url:string description:text price_in_cents:integer
```

Now update the model for this to make sure that we always have a name and a price (in cents):

```rb
# app/models/product.rb
class Product < ApplicationRecord
  validates :name, :price_in_cents, presence: true

  def price
    price_in_cents / 100
  end  
end
```

Keep in mind that we could have stored the price as a float or decimal. However, we store `price_in_cents` to prevent rounding errors. To ensure that we don't have issues viewing the site, run a database migration:

```sh
$ rake db:migrate
```

Now let's create some dummy data to work with. Add this code to your `db/seeds.rb`:

```rb
# db/seeds.rb

Product.create!(name: "First Product", price_in_cents: 100)
Product.create!(name: "Second Product", price_in_cents: 200)
Product.create!(name: "Third Product", price_in_cents: 300)

```

Now run the database seeds:

```sh
$ rake db:seed
```

This should load up the database with three different products, at $1, $2, and $3, respectively.

The next step is updating the main page. We'll be updating is `app/views/products/index.html.erb` to show our gallery. In the next few sections, we'll create some React components that will go on this page.

### Adding a `ProductCard` and `ProductRow` component

The first React component we will build is the `ProductCard`, which is our term for an individual product in the gallery. Go to `client/bundles` and create a new directory called `WebStore/components`:

```sh
$ mkdir -p client/bundles/WebStore/components
```

While you're at it, you can remove the `HelloWorld` directory, which we no longer need.

```sh
$ rm -rf client/bundles/HelloWorld
```

Next create a new file called `ProductCard.jsx`. Here's my implementation of the `ProductCard` component:

```js
// client/bundles/WebStore/components/ProductCard.jsx

import React from 'react';

export class ProductCard extends React.Component {
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
              This is a product description. It describes the product and gives the reader a reason to buy it.
            </p>
            <div className="d-flex justify-content-between align-items-center">
              <div className="btn-group">
                <button type="button" className="btn btn-sm btn-outline-secondary">
                  Add to Cart
                </button>
              </div>
              <div>{amount}</div>
            </div>
          </div>
        </div>
      </div>
    );
  }
}
```

Almost all of the CSS classes used here are available via Twitter Bootstrap. The only one that is not is `product-image`, which isn't an image at the moment, but serves as a placeholder for now. This is where I defined the CSS class:

```css
/* app/assets/stylesheets/application.scss */
@import "bootstrap";

/* Adding this new CSS class */
.product-image {
  height: 200px;
  background-color: gray;
  color: white;
}
```

If you know Twitter Bootstrap well, you may notice my use of `col-md-4`, which only works in the context of `row`. A thin container component to fix this is a `ProductRow`:

```js
/* client/bundles/WebStore/components/ProductRow.jsx */

import React from 'react';

import { ProductCard } from './ProductCard.jsx';

export class ProductRow extends React.Component {
  render() {
    return (
      <div className="row">
        {this.props.products.map((props) =>
          <ProductCard key={props.id} {...props}/>)}
      </div>
    );
  }
}
```

At the moment, using `col-md-4` means that `ProductRow` can only handle 3 products at a time. In the future, we should update this to use CSS Flexbox so an arbitrary number of products.

### Adding a `SiteHeader` component

We have component-ized the product gallery. However, we will eventually want to show the user the number of products in their cart. In that situation, we'll need to have a dynamic navigation bar, with a place to access their current cart. This is a `SiteHeader` component that will eventually update the current number of products in our cart. Place this inside `client/bundles/WebStore/components/SiteHeader.jsx`:

```js
/* client/bundles/WebStore/components/SiteHeader.jsx */

import React from 'react';

export class SiteHeader extends React.Component {
  render() {
    return (
      <header>
        <div className="navbar navbar-dark bg-dark box-shadow">
          <a href="/products" className="navbar-brand d-flex align-items-center">
            <strong>Web Store</strong>
          </a>
          <span className="navbar-brand">
            In Cart: 0
          </span>
        </div>
      </header>
    );
  }
}
```

After this component, we can set up our JavaScript bundle for Rails to create on each page refresh.

### Adding the JavaScript bundle

In order to render JavaScript to the page, Rails needs to be aware of which particular files we'll be using. Fortunately Webpack handles a lot of that work for us. To do this, we'll use `import` to load them, and then use `ReactOnRails.register` to inform the `react_on_rails` gem that these classes are available as components. Add the following file as `client/packs/web-store-bundle.js`:

```js
/* client/packs/web-store-bundle.js */

import ReactOnRails from 'react-on-rails';

import { ProductRow } from '../bundles/WebStore/components/ProductRow.jsx';
import { SiteHeader } from '../bundles/WebStore/components/SiteHeader.jsx';

ReactOnRails.register({
  SiteHeader,
  ProductRow
});
```

And while we're here, we can remove the extra files generated by `react_on_rails`:

```sh
$ rm client/packs/application.js
$ rm client/packs/hello_react.jsx
$ rm client/packs/hello-world-bundle.js
```

### Updating the view and the layout

The main page should be updated to use the `ProductRow` component. Replace the contents of `app/view/products/index.html.erb` to look like this:

```html
<!-- app/views/products/index.html.erb -->

<section class="jumbotron text-center">
  <div class="container">
    <h1 class="jumbotron-heading">Product Gallery</h1>
  </div>
</section>

<div class="container">
  <%= react_component("ProductRow", props: {products: Product.all.as_json(methods: :price)}) %>
</div>
```

The above call to `react_component` will provide the list of existing products as JSON. As stated before, the design only supports three products at a time. We could use Flexbox, but another possible, but less optimal, solution is to split the `Product.all` array into subarrays of size three.

Now go to `app/views/layouts/application.html.erb` and update it to look like this:

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
    <%= react_component("SiteHeader") %>

    <main role="main">
      <%= yield %>
    </main>
  </body>
</html>
```

After updating the layout, we have one last step before running the server. Let's make the root URL point to our products:

```rb
# config/routes.rb
Rails.application.routes.draw do
  resources :products
  root to: 'products#index'
end
```

Now you can start up your Rails server:

```sh
$ rails server
```

If you did everything correctly, you can go to the main product page (`http://localhost:3000/`).  This is what the page will look like:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/ckr0zddcgzu998kjowq1.png)


# Conclusion

For our first post, we learned about integrating React on Ruby on Rails to create a product gallery. We used Twitter Bootstrap and custom React components to show three products that we have stored in our database. We used JSON serialization to send the product data from the Rails back-end to the React component front-end. At the moment the "Add to Cart" buttons do not work. In the next post we'll be setting those up. See you then!
