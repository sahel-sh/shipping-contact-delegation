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
remaining data itself. A simplier alternative is to only show the payment handlers that can handle requested information in the Payment Instruments list. 

# Enable Delegations
Payment Handlers should specify which user infromation they can provide during registration.
```idl
enum PaymentDelegation {
  "shippingAddress",
  "payerName",
  "payerPhone",
  "payerEmail",
};
interface PaymentManager {
  …
  Promise<void> enableDelegations(FrozenArray<PaymentDelegation> delegations);
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
    readonly attribute object?   paymentOptions;
    readonly attribute FrozenArray<PaymentShippingOption>?    shippingOptions;
};

dictionary PaymentRequestEventInit : ExtendableEventInit {
    … 
    PaymentOptions paymentOptions;
    sequence<PaymentShippingOption> shippingOptions;
};
```
# Payment Handler Response
The payment handler should provide required information in its response. If shipping is requested the response should include
the shipping option id in addition to the shipping address. Then the browser will validate the response (country format, shipping address Id, non-empty strings for required payer information, etc) and send it the merchant.
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
# Shipping Address/Option Change Events
Payment handlers which support shipping delegation should be able to fire [shipping[address|option]change](https://w3c.github.io/payment-request/#summary) events to let the merchant know that the user has selected a different shipping address/option. The browser will then get the
updated request details from merchant and forward it to the payment handler after redacting [displayItems](https://w3c.github.io/payment-request/#dom-paymentdetailsbase-displayitems). We propose to change [PaymentMethodChangeResponse](https://w3c.github.io/payment-handler/#the-paymentmethodchangeresponse) into a generic dictionary used for forwarding merchant's responses to [payment method|shipping address\shipping option] events to payment handlers.
```idl
interface PaymentRequestEvent : ExtendableEvent {
    … 
    Promise<PaymentRequestDetailsUpdate?> changeShippingAddress(PaymentAddressInit shippingAddress);
    Promise<PaymentRequestDetailsUpdate?> changeShippingOption(DOMString shippingOption);
};

dictionary PaymentRequestDetailsUpdate {
    … 
    FrozenArray<PaymentShippingOption> shippingOptions;
    … 
    AddressErrors shippingAddressErrors;
};
```
# Edit: Skip the Payment Sheet
When the merchant calls paymentRequest.show(), the browser shows the "Payment Sheet" to let the user select/enter payment instrument, as well as shipping address/option and/or contact information.

Under certain conditions Chrome decides to skip showing the payment sheet. Instead, goto the payment handler window directly to reduce the checkout steps. Currently asking for shipping address or contact information forces showing the "Payment Sheet" since the browser is responsible for providing this information. We propose to skip the payment sheet in cases where shipping address/contact info is required and the available payment handler can provide the merchant required information (other conditions are unchanged).

# Edit: Just In Time (JIT) Installation
Under certain conditions Chrome might show an uninstalled payment handler in available payment instruments list, and install the payment handler later during the checkout if the user selects to proceed with it. Since payment handlers declare supported delegations during their registration, the browser does not know whether or not a JIT payment handler handles shipping or contact information while showing the payment sheet. To address this, payment handlers can specify their supported delegations in [Web App Manifest](https://www.w3.org/TR/appmanifest/) which is parsed before installation.

