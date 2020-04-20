# Integration with Stripe


## Sign up for Stripe


## Get your publishable key (needed for later)



## Adding Stripe.js React elements

This part of the tutorial relies on Stripe.js React elements. For reference, you can check their [official documentation page](https://stripe.com/docs/stripe-js/react). In order to use the Stripe.js React elememnts, we need to add the necessary NPM packages. Run the following to install them:

```sh
$ npm install --save @stripe/react-stripe-js @stripe/stripe-js
```

Next we'll create the `CheckoutForm` component. This is really just the `Card-Minimal.js` example provided by Stripe, [available on their GithHub page](https://github.com/stripe/react-stripe-js/blob/9fe1a5473cd1125fcda4e01adb6d6242a9bae731/examples/hooks/0-Card-Minimal.js#L9-L64), but with some modifications for this tutorial:

```js
// client/bundles/WebStore/components/CheckoutForm.jsx

import React from 'react';
import { loadStripe } from '@stripe/stripe-js';
import {
  CardElement, Elements, useStripe, useElements
} from '@stripe/react-stripe-js';
import { BillingInfo } from './BillingInfo.jsx';

const StripeCheckoutForm = () => {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (event) => {
    event.preventDefault();

    if (!stripe || !elements) {
      return;
    }

    const cardElement = elements.getElement(CardElement);

    const {error, paymentMethod} = await stripe.createPaymentMethod({
      type: 'card',
      card: cardElement,
    });

    if (error) {
      console.log('[error]', error);
    } else {
      console.log('[PaymentMethod]', paymentMethod);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <BillingInfo/>
      <hr className="mb-4"/>
      <h4 className="mb-3">Payment</h4>
      <div className="mb-12">
        <label htmlFor="card-element">
          Credit or Debit Card
        </label>
        <div id="card-element" className="form-control" style={{height: '2.4em', paddingTop: '.7em', marginBottom: '1em'}}>
          <CardElement />
        </div>
      </div>

      <button className="btn btn-primary btn-lg btn-block" type="submit">
        Confirm and Pay
      </button>
    </form>
  );
};

export const CheckoutForm = () => {
  const stripePromise = loadStripe('<YOUR STRIPE PUBLISHABLE ID HERE>');

  return (
    <Elements stripe={stripePromise}>
      <StripeCheckoutForm />
    </Elements>
  );
};
```

The `BillingInfo` component was referenced in the above component. The definition for it is below. It is virtually a conversion of the previous HTML template into a React component:

```js
// client/bundles/WebStore/components/BillingInfo.jsx

import React from 'react';

export const BillingInfo = () => {
  return (
    <>
      <h4 className="mb-3">Billing address</h4>
      <div className="mb-3">
        <label htmlFor="name">Name</label>
        <input type="text" className="form-control" id="name" placeholder="" defaultValue="" required=""/>
        <div className="invalid-feedback">
          Name is required.
        </div>
      </div>
      <div className="mb-3">
        <label htmlFor="email">Email</label>
        <input type="email" className="form-control" id="email" placeholder="you@example.com"/>
        <div className="invalid-feedback">
          Please enter a valid email address for shipping updates.
        </div>
      </div>
      <div className="mb-3">
        <label htmlFor="address">Address</label>
        <input type="text" className="form-control" id="address" placeholder="1234 Main St" required=""/>
        <div className="invalid-feedback">
          Please enter your shipping address.
        </div>
      </div>
      <div className="mb-3">
        <label htmlFor="address2">Address 2 <span className="text-muted">(Optional)</span></label>
        <input type="text" className="form-control" id="address2" placeholder="Apartment or suite"/>
      </div>
      <div className="row">
        <div className="col-md-5 mb-3">
          <label htmlFor="country">Country</label>

          <select className="custom-select d-block w-100" id="country" required="" defaultValue="US">
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
            <option value="US">United States</option>
          </select>

          <div className="invalid-feedback">
            Please select a valid country.
          </div>
        </div>
        <div className="col-md-4 mb-3">
          <label htmlFor="state">State</label>

          <input type="text" name="state" className="form-control" id="zip" placeholder="" required=""/>

          <div className="invalid-feedback">
            Please provide a valid state.
          </div>
        </div>
        <div className="col-md-3 mb-3">
          <label htmlFor="zip">Zip</label>
          <input type="text" className="form-control" id="zip" placeholder="" required=""/>
          <div className="invalid-feedback">
            Zip code required.
          </div>
        </div>
      </div>
    </>
  );
}
```

Next update your JavaScript pack to include the new `CheckoutForm` component and register it with React on Rails:

```js
// client/packs/web-store-bundle.js

import ReactOnRails from 'react-on-rails';

import { SiteHeader } from '../bundles/WebStore/components/SiteHeader.jsx';
import { ProductRow } from '../bundles/WebStore/components/ProductRow.jsx';
import { ShoppingCartList } from '../bundles/WebStore/components/ShoppingCartList.jsx';
import { CheckoutForm } from '../bundles/WebStore/components/CheckoutForm.jsx';

ReactOnRails.register({
  SiteHeader,
  ProductRow,
  ShoppingCartList,
  CheckoutForm
});
```

Now that we've converted the billing form fields and payment info fields into a React component, we can remove the partials:

```sh
$ rm app/views/checkout/_billing_info.html.erb
$ rm app/views/checkout/_payment_info.html.erb
```

Now you can change the checkout page to look like this:

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
      <%= react_component("CheckoutForm") %>
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

You can fill out the credit car number with a test one, provided by Stripe. I've done that in the screenshot below using the test number "4242 4242 4242 4242". The expiration date, CVC, and billing zip code should not matter for the test account, so long as it's not expired.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/odxg9krzvt8mwkuwr8ad.png)

## Payment Success

After submitting "Confirm and Pay", nothing happens. We want to redirect the customer to a confirmation page. Towards that end, let us set up the `/checkout/complete` page to look like this:

```html
<!-- app/views/checkout/complete.html.erb -->

<section class="jumbotron text-center">
  <div class="container">
    <h1 class="jumbotron-heading">Your Order has Been Processed!</h1>
  </div>
</section>

<div class="container">
  <% current_customer.shopping_cart.shopping_cart_items.each do |item| %>
    <div class="row">
      <div class="col-2"></div>

      <div class="col-6">
        <h4><%= item.product.name %></h4>
      </div>

      <div class="col-2">
        <h4>$<%= item.product.price %></h4>
      </div>
    </div>
  <% end %>

  <div class="row">
    <div class="col-2"></div>

    <div class="col-6">
      <h4>Total:</h4>
    </div>

    <div class="col-2">
      <h4>$<%= current_customer.shopping_cart.total_price %></h4>
    </div>
  </div>

  <hr className="mb-12"/>

  <div class="row align-items-center">
    <div class="col-2"></div>

    <div class="col-10">
      <h3>
        Thank you and please come again!
      </h3>
    </div>
  </div>
</div>
```

For convenience, I added a few more methods the models. In this case, one for the `ShoppingCart`, namely `total_price`:

```rb
# app/models/shopping_cart.rb

class ShoppingCart < ApplicationRecord
  belongs_to :customer
  has_many :shopping_cart_items

  # New convenience method
  def total_price
    shopping_cart_items.map(&:price).sum
  end
end
```

And another for `ShoppingCartItem`, the `price` method which takes into account the quantity as well:

```rb
# app/models/shopping_cart_item.rb

class ShoppingCartItem < ApplicationRecord
  belongs_to :shopping_cart
  belongs_to :product
  validates :quantity, presence: true

  def price
    product.price * quantity
  end
end
```


We can update the `CheckoutForm` component to redirect to `/checkout/complete` by changing the line mentioned below:

```js
// client/bundles/WebStore/components/CheckoutForm.jsx

import React from 'react';
import { loadStripe } from '@stripe/stripe-js';
import { CardElement, Elements, useStripe, useElements } from '@stripe/react-stripe-js';
import { BillingInfo } from './BillingInfo.jsx';

const StripeCheckoutForm = () => {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (event) => {
    event.preventDefault();

    if (!stripe || !elements) {
      return;
    }

    const cardElement = elements.getElement(CardElement);

    const {error, paymentMethod} = await stripe.createPaymentMethod({
      type: 'card',
      card: cardElement,
    });

    if (error) {
      console.log('[error]', error);
    } else {
      /*************************************************************
       This is the only line we're changing to redirect the customer
       *************************************************************/
      window.location = '/checkout/complete';
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <BillingInfo/>
      <hr className="mb-4"/>
      <h4 className="mb-3">Payment</h4>
      <div className="mb-12">
        <label htmlFor="card-element">
          Credit or Debit Card
        </label>
        <div id="card-element" className="form-control" style={{height: '2.4em', paddingTop: '.7em', marginBottom: '1em'}}>
          <CardElement />
        </div>
      </div>

      <button className="btn btn-primary btn-lg btn-block" type="submit">
        Confirm and Pay
      </button>
    </form>
  );
};

export const CheckoutForm = () => {
  const stripePromise = loadStripe('<YOUR STRIPE PUBLISHABLE ID HERE>');

  return (
    <Elements stripe={stripePromise}>
      <StripeCheckoutForm />
    </Elements>
  );
};
```

When the payment is successful, you should see a page like this:

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/3kna0zkyp6lyiv1bze6i.png)


## Conclusion

What's left? We can add functionality for when payment fails. We can email the customer a receipt. We can offer the customer a chance to create an account using their billing information. There's lots that this tutorial doesn't touch on that can be expanded here. Hopefully you've foundt his helpful as a starting point and continue your journey making a full featured web store!
