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

# Edit: Security And Privacy Self-Assessment
1- What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?
>We do not share any extra information about the user with merchant's origin, but the source of the shipping address and payer's contact information changes from the browser to the user selected Payment Handler. The browser will still apply the same validation/redacting to the data provided by the payment handler before passing it to merchant's origin. The only extra piece of information that we share with Payment Handlers is whether or not the transaction requires shipping address/ or payer's contact info. If shipping address is requested, the browser also shares merchant provided shipping options with payment handler, so that the payment handler can ask the user to select their desired shipping option (e.g, same day, express, standard, etc.) We only share this information with the PH which specifies that it will collect shipping address and contact information from the user.

2- Is this specification exposing the minimum amount of information necessary to power the feature?
> The only extra transaction information exposed to the payment handler is boolean bits indicating whether or not the merchant has asked for shipping address or payer's contact information (name, phone, email), as well as merchant provided shipping options when shipping is requested. This information is shared with payment handlers that are capable of providing the merchant required information and only after the user selects the payment handler to complete the transaction.

3- How does this specification deal with personal information or personally-identifiable information or information derived thereof?


4- How does this specification deal with sensitive information?
>It is unchanged. i.e. The browser still validates/redacts shipping address/ contact information regardless of whether it comes from the browser or payment handler before sending it to the merchant.

5- Does this specification introduce new state for an origin that persists across browsing sessions?
>No, it does not.

6- What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?
>None

7- Does this specification allow an origin access to sensors on a user’s device
>No

8- What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts
> The user data is identical to [PaymentResponse](https://www.w3.org/TR/payment-request/#dom-paymentresponse) from Payment Request API.

9- Does this specification enable new script execution/loading mechanisms?
>No

10- Does this specification allow an origin to access other devices?
>No

11- Does this specification allow an origin some measure of control over a user agent’s native UI?
>No

12- What temporary identifiers might this specification create or expose to the web?
>None

13- How does this specification distinguish between behavior in first-party and third-party contexts?


14- How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
> Unchanged, identical to Payment Request and Payment Handler APIs.

15- Does this specification have a "Security Considerations" and "Privacy Considerations" section?
> Not yet, it will, once the spec draft is ready.

16- Does this specification allow downgrading default security characteristics?
> No


