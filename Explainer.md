# Introduction
Merchants can ask for a shipping address or payer's contact information specifying `paymentOptions` in 
[Payment Request](https://w3c.github.io/payment-request/#dom-paymentoptions). Today the data collection is always handled by
the browser. Delegating this data collection to payment handlers can lead to better user experience since the payment
app may have more accurate information than the browser. Additionally new user onboarding flow would have less friction since
shipping/contact and payment information collection won't be split between the browser and the payment provider.

# Proposal
Payment handler will let the browser know that it can provide shipping address and/or payer's contact(name, phone, email)
information during installation. During checkout, the browser will include the merchant specified `paymentOptions` in the
[PaymentRequestEvent](https://w3c.github.io/payment-handler/#the-paymentrequestevent) sent to the
payment handler. Then, it will collect shipping and/or payer's contact information from the handlers'
[response](https://w3c.github.io/payment-handler/#paymenthandlerresponse-dictionary).
For compatibility purposes we propose partial delegation whenever needed. i.e. If the selected payment instrument can handle a
subset of the required paymentOptions, the browser will only send that subset to the payment handler and provide/collect the
remaining data itself.

# Enable Delegations
The payment handler should specify which user infromation they can handle.
```idl
enum PaymentDelegation {
  "shippingAddress",
  "payerName",
  "payerPhone",
  "payerEmail",
};
interface PaymentManager {
  …
  [CallWith=ScriptState] Promise<void> enableDelegations(FrozenArray<PaymentDelegation> delegations);
};
```

```javascript
// Payment Handler
async function install() {
  try {
    await navigator.serviceWorker
      .register('ph_sw.js');
    registration = await navigator
      .serviceWorker.ready;
    if (!registration.paymentManager) {
      return 'PaymentManager API not found.';
    }
    await registration.paymentManager
      .enableDelegations(
        ['shippingAddress',
          'payerName', 'payerPhone',
          'payerEmail'
        ]);
    … 
    return 'success';
  } catch (e) {
    return e.toString();
  }
}
```

# Browser Passes on paymentOptions
The browser should include merchant requested paymentOptions in PaymentRequestEvent. If shipping is requested the
browser should include merchant provided shippingOptions as well.

```idl
interface PaymentRequestEvent : ExtendableEvent {
    … 
    [CallWith=ScriptState] readonly attribute object?   paymentOptions;
    readonly attribute FrozenArray<PaymentShippingOption>?    shippingOptions;
}

dictionary PaymentRequestEventInit : ExtendableEventInit {
    … 
    PaymentOptions paymentOptions;
    sequence<PaymentShippingOption> shippingOptions;
};
```
# Payment Handler Response
The payment handler should provide required information in its response. If shipping is requested the response should include
a shipping option id in addition to the shipping address.
```idl
dictionary PaymentHandlerResponse {
    … 
    DOMString? payerName;
    DOMString? payerEmail;
    DOMString? payerPhone;
    PaymentAddressInit shippingAddress;
    DOMString? shippingOption;
};
```
