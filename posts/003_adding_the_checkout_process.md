## Adding the checkout process

When a user is ready to pay for their shopping cart, we will show them a checkout page. It will be loosely based on the Bootstrap example checkout page:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/9cdiumotr3u4hv1m64oj.png)

We're going to create a new controller specifically for this purpose:

```sh
$ rails generate controller Checkout index complete
```

This will generate a URL that looks like this: `http://localhost:3000/checkout/index`. The word `index` seems unnecessary. We want one that looks like this: `http://localhost:3000/checkout`. Let's update the routes file to fix that:

```rb
# config/routes.rb

Rails.application.routes.draw do
  get 'checkout', to: 'checkout#index'
  get 'checkout/complete'
  post 'shopping_cart/add'
  resources :products
  root to: 'products#index'
end
```

The initial version of checkout form is very complex, as it contains the billing information form as well as the payment form. Let's break it down into parts, to prevent confusion. The first part to creating this page is to set up the form itself. It will not be renderable right away because some of the files are defined after. Here's the basic layout of the checkout page:

```html
<!-- app/views/checkout/index.html.erb -->

<section class="jumbotron text-center">
  <div class="container">
    <h1 class="jumbotron-heading">Checkout</h1>
  </div>
</section>

<div class="container">
  <div class="row">
    <div class="col-md-8">
      <h4 class="mb-3">Billing address</h4>

      <form class="needs-validation" novalidate="">

        <%= render partial: 'checkout/billing_info' %>

        <hr class="mb-4">

        <h4 class="mb-3">Payment</h4>

        <%= render partial: 'checkout/payment_info' %>

        <hr class="mb-4">

        <button class="btn btn-primary btn-lg btn-block" type="submit">
          Confirm and Pay
        </button>
      </form>
    </div>
  </div>
</div>
```

The next part is the billing address, which is rather long, but only because there are many fields. The country list also makes this partial seem pretty long. The partial is below:

```html
<!-- app/views/checkout/_billing_info.html.erb -->

<div class="mb-3">
  <label for="name">Name</label>
  <input type="text" class="form-control" id="name" placeholder="" value="" required="">
  <div class="invalid-feedback">
    Name is required.
  </div>
</div>

<div class="mb-3">
  <label for="email">Email</label>
  <input type="email" class="form-control" id="email" placeholder="you@example.com">
  <div class="invalid-feedback">
    Please enter a valid email address for shipping updates.
  </div>
</div>

<div class="mb-3">
  <label for="address">Address</label>
  <input type="text" class="form-control" id="address" placeholder="1234 Main St" required="">
  <div class="invalid-feedback">
    Please enter your shipping address.
  </div>
</div>

<div class="mb-3">
  <label for="address2">Address 2 <span class="text-muted">(Optional)</span></label>
  <input type="text" class="form-control" id="address2" placeholder="Apartment or suite">
</div>

<div class="row">
  <div class="col-md-5 mb-3">
    <label for="country">Country</label>

    <select class="custom-select d-block w-100" id="country" required="">
      <option value="AU">Australia</option>
      <option value="AT">Austria</option>
      <option value="BE">Belgium</option>
      <option value="BR">Brazil</option>
      <option value="CA">Canada</option>
      <option value="CN">China</option>
      <option value="DK">Denmark</option>
      <option value="FI">Finland</option>
      <option value="FR">France</option>
      <option value="DE">Germany</option>
      <option value="HK">Hong Kong</option>
      <option value="IE">Ireland</option>
      <option value="IT">Italy</option>
      <option value="JP">Japan</option>
      <option value="LU">Luxembourg</option>
      <option value="MY">Malaysia</option>
      <option value="MX">Mexico</option>
      <option value="NL">Netherlands</option>
      <option value="NZ">New Zealand</option>
      <option value="NO">Norway</option>
      <option value="PT">Portugal</option>
      <option value="SG">Singapore</option>
      <option value="ES">Spain</option>
      <option value="SE">Sweden</option>
      <option value="CH">Switzerland</option>
      <option value="GB">United Kingdom</option>
      <option value="US" selected="selected">United States</option>
    </select>

    <div class="invalid-feedback">
      Please select a valid country.
    </div>
  </div>

  <div class="col-md-4 mb-3">
    <label for="state">State</label>

    <input type="text" name="state" class="form-control" id="zip" placeholder="" required="">

    <div class="invalid-feedback">
      Please provide a valid state.
    </div>
  </div>

  <div class="col-md-3 mb-3">
    <label for="zip">Zip</label>
    <input type="text" class="form-control" id="zip" placeholder="" required="">
    <div class="invalid-feedback">
      Zip code required.
    </div>
  </div>
</div>
```

The next partial is for the payment form. This part will soon be replaced when we integrate Stripe's React components, so think of it as a placeholder for now:

```html
<!-- app/views/checkout/_payment_info.html.erb -->

<div class="row">
  <div class="col-md-6 mb-3">
    <label for="cc-name">Name on card</label>
    <input type="text" class="form-control" id="cc-name" placeholder="" required="">
    <small class="text-muted">Full name as displayed on card</small>
    <div class="invalid-feedback">
      Name on card is required
    </div>
  </div>
  <div class="col-md-6 mb-3">
    <label for="cc-number">Credit card number</label>
    <input type="text" class="form-control" id="cc-number" placeholder="" required="">
    <div class="invalid-feedback">
      Credit card number is required
    </div>
  </div>
</div>
<div class="row">
  <div class="col-md-3 mb-3">
    <label for="cc-expiration">Expiration</label>
    <input type="text" class="form-control" id="cc-expiration" placeholder="" required="">
    <div class="invalid-feedback">
      Expiration date required
    </div>
  </div>
  <div class="col-md-3 mb-3">
    <label for="cc-expiration">CVV</label>
    <input type="text" class="form-control" id="cc-cvv" placeholder="" required="">
    <div class="invalid-feedback">
      Security code required
    </div>
  </div>
</div>
```

After adding all this HTML, verify that the page looks something like this:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/mnsrrf6vftewpkmdsnvc.png)

We have left the right side of the page blank for now, to make room for the list of products that the customer is about to purchase.

## Adding the Shopping Cart to the right side

While we have not yet implemented the payment process, we can still continue to flesh out this page. Let's create a React component to show the list of products that are about to be purchased. Create a new component under `client/bundles/WebStore/components/ShoppingCartList.jsx`:

```js
/* client/bundles/WebStore/components/ShoppingCartList.jsx */

import React from 'react';

export class ShoppingCartList extends React.Component {
  formatPrice(price) {
    return price.toLocaleString('en-US', {
      style: 'currency',
      currency: 'USD'
    });
  }

  renderListItem(item) {
    const description = item.product.description || "Brief description";

    return (
      <li key={item.id} className="list-group-item d-flex justify-content-between lh-condensed">
        <div>
          <h6 className="my-0">{item.product.name}</h6>
          <small className="text-muted">{description}</small>
        </div>
        <span className="text-muted">
          {this.formatPrice(item.product.price)}
        </span>
      </li>
    );
  }

  render() {
    const count = this.props.shopping_cart_items.length;

    const totalPrice =
      this.props.
        shopping_cart_items.map(item => item.product.price).
        reduce((t, s) => t + s, 0);

    return (
      <>
        <h4 className="d-flex justify-content-between align-items-center mb-3">
          <span className="text-muted">Your cart</span>
          <span className="badge badge-secondary badge-pill">
            {count}
          </span>
        </h4>
        <ul className="list-group mb-3">
          {this.props.shopping_cart_items.map(item =>
            this.renderListItem(item))}
          <li className="list-group-item d-flex justify-content-between">
            <span>Total (USD)</span>
            <strong>{this.formatPrice(totalPrice)}</strong>
          </li>
        </ul>
      </>
    );
  }
}
```

Now update the pack to include the new component:

```js
// client/packs/web-store-bundle.js

import ReactOnRails from 'react-on-rails';

import { SiteHeader } from '../bundles/WebStore/components/SiteHeader.jsx';
import { ProductRow } from '../bundles/WebStore/components/ProductRow.jsx';
import { ShoppingCartList } from '../bundles/WebStore/components/ShoppingCartList.jsx';

ReactOnRails.register({
  SiteHeader,
  ProductRow,
  ShoppingCartList
});
```

And then add the new component to the checkout page:

```html
<!-- app/views/checkout/index.html.erb -->

<section class="jumbotron text-center">
  <div class="container">
    <h1 class="jumbotron-heading">Checkout</h1>
  </div>
</section>

<div class="container">
  <div class="row">
    <div class="col-md-8">
      <h4 class="mb-3">Billing address</h4>

      <form class="needs-validation" novalidate="">
        <%= render partial: 'checkout/billing_info' %>

        <hr class="mb-4">

        <h4 class="mb-3">Payment</h4>

        <%= render partial: 'checkout/payment_info' %>

        <hr class="mb-4">

        <button class="btn btn-primary btn-lg btn-block" type="submit">
          Confirm and Pay
        </button>
      </form>
    </div>

    <div class="col-md-4 mb-4">
      <%= react_component("ShoppingCartList", props: {
        shopping_cart_items: current_customer.
          shopping_cart.shopping_cart_items.as_json(include: {
            product: {methods: :price}
          })
        })
      %>
    </div>    
  </div>
</div>
```

After adding the shopping cart list to the right side of the form, you should see your page looks like this:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/wsb2jj60gqbh9xdhw3ev.png)

## Conclusion

Talk about the conclusion

....

TODO TODO TODO TODO
