# Magento 2 sample payment method gateway


[**Magento 2 Create Payment Method**](https://www.mageplaza.com/magento-2-create-payment-method/) proves that store admin has rights to generate as many payment methods as they need when your store is based on Magento 2 platform, an great era of ecommerce architecture. Depending on the customer's requirement, you probably plug it in your list of the existing payment method. The additional payment methods surely bring the diversity of customer choice when they proceed to checkout on your site. On the other's hands, multiple payment method is the great strategy to reach out the global marketplace.

In the tutorial, you will learn how to create own Payment Gateway integration in Magento 2 stores. After launching the new payment methods, you will find and configure it according the path `Admin panel > Stores > Settings > Configuration > Sales > Payment Methods`. There, admin possibly assigns a payment method to specific shipping method, this means they will work in pairs when enabling.


## Step 1: Create payment method module

1. Create file [registration.php](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/registration.php)

``` php
<?php
\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'Mageplaza_Payment',
    __DIR__
);
```

2. Declare `module.xml` file

``` xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Mageplaza_Payment" setup_version="0.1.0">
        <sequence>
            <module name="Magento_Sales"/>
            <module name="Magento_Payment"/>
            <module name="Magento_Checkout"/>
            <module name="Magento_Directory" />
            <module name="Magento_Config" />
        </sequence>
    </module>
</config>
```

This module have to run after the Magento_Sales, Magento_Payment, Magento_Checkout, Magento_Directory, and Magento_Config. So we add depends (sequence) them like above block of code.

## Step 2: Declare payment method module

1. Now we create file `payment.xml` file in `etc` folder [etc/payment.xml](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/etc/payment.xml)

``` xml
<?xml version="1.0" ?>
<payment xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Payment:etc/payment.xsd">
    <groups>
        <group id="offline">
            <label>Offline Payment Methods</label>
        </group>
    </groups>
    <methods>
        <method name="simple">
            <allow_multiple_address>1</allow_multiple_address>
        </method>
    </methods>
</payment>
```

2. Create `config.xml` file in `etc` folder.

``` xml
<?xml version="1.0" ?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Store:etc/config.xsd">
    <default>
        <payment>
            <simple>
                <active>1</active>
                <model>Mageplaza\Payment\Model\Payment\Simple</model>
                <order_status>pending</order_status>
                <title>Simple</title>
                <allowspecific>0</allowspecific>
                <group>Offline</group>
            </simple>
        </payment>
    </default>
</config>
```

In this file, we declare Model of this payment method, we calle `Simple` model.

3. Create Simple model file

Create file [Model/Payment/Simple.php](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/Model/Payment/Simple.php)

``` php
<?php
namespace Mageplaza\Payment\Model\Payment;
class Simple extends \Magento\Payment\Model\Method\Cc
{
    protected $_isGateway = true;
    protected $_canCapture = true;
    protected $_canCapturePartial = true;
    protected $_canRefund = true;
    protected $_canRefundInvoicePartial = true;
    protected $_stripeApi = false;
    protected $_countryFactory;
    protected $_minAmount = null;
    protected $_maxAmount = null;
    protected $_supportedCurrencyCodes = array('USD');
    protected $_debugReplacePrivateDataKeys
        = ['number', 'exp_month', 'exp_year', 'cvc'];
    public function __construct(\Magento\Framework\Model\Context $context,
        \Magento\Framework\Registry $registry,
        \Magento\Framework\Api\ExtensionAttributesFactory $extensionFactory,
        \Magento\Framework\Api\AttributeValueFactory $customAttributeFactory,
        \Magento\Payment\Helper\Data $paymentData,
        \Magento\Framework\App\Config\ScopeConfigInterface $scopeConfig,
        \Magento\Payment\Model\Method\Logger $logger,
        \Magento\Framework\Module\ModuleListInterface $moduleList,
        \Magento\Framework\Stdlib\DateTime\TimezoneInterface $localeDate,
        \Magento\Directory\Model\CountryFactory $countryFactory,
        array $data = array()
    ) {
        parent::__construct(
            $context, $registry, $extensionFactory, $customAttributeFactory,
            $paymentData, $scopeConfig, $logger, $moduleList, $localeDate, null,
            null, $data
        );
        $this->_countryFactory = $countryFactory;
        $this->_minAmount = $this->getConfigData('min_order_total');
        $this->_maxAmount = $this->getConfigData('max_order_total');
    }
    /**
     * Authorize payment abstract method
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     * @api
     * @SuppressWarnings(PHPMD.UnusedFormalParameter)
     */
    public function authorize(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canAuthorize()) {
            throw new \Magento\Framework\Exception\LocalizedException(__('The authorize action is not available.'));
        }
        return $this;
    }
    /**
     * Capture payment abstract method
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     * @api
     * @SuppressWarnings(PHPMD.UnusedFormalParameter)
     */
    public function capture(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canCapture()) {
            throw new \Magento\Framework\Exception\LocalizedException(__('The capture action is not available.'));
        }
        return $this;
    }
    /**
     * Refund specified amount for payment
     *
     * @param \Magento\Framework\DataObject|InfoInterface $payment
     * @param float $amount
     * @return $this
     * @throws \Magento\Framework\Exception\LocalizedException
     * @api
     * @SuppressWarnings(PHPMD.UnusedFormalParameter)
     */
    public function refund(\Magento\Payment\Model\InfoInterface $payment, $amount)
    {
        if (!$this->canRefund()) {
            throw new \Magento\Framework\Exception\LocalizedException(__('The refund action is not available.'));
        }
        return $this;
    }
}
```

This model includes basic functions such as 
- `authorize()`: Authorize the payment e.g: card
- `capture()`: Capture money from a customer
- `refund()`: Chargeback money to the customer.


## Step 3: Display payment method in checkout page

In previous steps, we talked about declare module, config and model files. Now we need to display this Simple payment method in checkout page.

1. Create layout file: [view/frontend/layout/checkout_index_index.xml](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/view/frontend/layout/checkout_index_index.xml)

``` xml
<?xml version="1.0" ?>
<pagepage layout="1column" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <body>
        <referenceBlock name="checkout.root">
            <arguments>
                <argument name="jsLayout" xsi:type="array">
                    <item name="components" xsi:type="array">
                        <item name="checkout" xsi:type="array">
                            <item name="children" xsi:type="array">
                                <item name="steps" xsi:type="array">
                                    <item name="children" xsi:type="array">
                                        <item name="billing-step" xsi:type="array">
                                            <item name="children" xsi:type="array">
                                                <item name="payment" xsi:type="array">
                                                    <item name="children" xsi:type="array">
                                                        <item name="renders" xsi:type="array">
                                                            <item name="children" xsi:type="array">
                                                                <item name="offline-payments" xsi:type="array">
                                                                    <item name="component" xsi:type="string">Magento_OfflinePayments/js/view/payment/offline-payments</item>
                                                                    <item name="methods" xsi:type="array">
                                                                        <item name="simple" xsi:type="array">
                                                                            <item name="isBillingAddressRequired" xsi:type="boolean">true</item>
                                                                        </item>
                                                                    </item>
                                                                </item>
                                                            </item>
                                                        </item>
                                                    </item>
                                                </item>
                                            </item>
                                        </item>
                                    </item>
                                </item>
                            </item>
                        </item>
                    </item>
                </argument>
            </arguments>
        </referenceBlock>
    </body>
</pagepage>
```


2. Create js files to load KO template in checkout page

- File: [/view/frontend/web/js/view/payment/simple.js](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/view/frontend/web/js/view/payment/simple.js)

``` js
define(
    [
        'uiComponent',
        'Magento_Checkout/js/model/payment/renderer-list'
    ],
    function (Component,
              rendererList) {
        'use strict';
        rendererList.push(
            {
                type: 'simple',
                component: 'Mageplaza_Payment/js/view/payment/method-renderer/simple-method'
            }
        );
        return Component.extend({});
    }
);
```

In this file, it call another component: `js/view/payment/method-renderer/simple-method`


Now we need to create `/view/frontend/web/js/view/payment/method-renderer/simple-method.js`

``` js
define(
    [
        'Magento_Checkout/js/view/payment/default'
    ],
    function (Component) {
        'use strict';
        return Component.extend({
            defaults: {
                template: 'Mageplaza_Payment/payment/simple'
            },
            getMailingAddress: function () {
                return window.checkoutConfig.payment.checkmo.mailingAddress;
            },
        });
    }
);
```


3. Finally, create KO template file [/view/frontend/web/template/payment/simple.html](https://github.com/mageplaza/magento-2-sample-payment-method/blob/master/view/frontend/web/template/payment/simple.html)

``` php
<div class="payment-method" data-bind="css: {'_active': (getCode() == isChecked())}">
    <div class="payment-method-title field choice">
        <input type="radio"
               name="payment[method]"
               class="radio"
               data-bind="attr: {'id': getCode()}, value: getCode(), checked: isChecked, click: selectPaymentMethod, visible: isRadioButtonVisible()"/>
        <label data-bind="attr: {'for': getCode()}" class="label"><span data-bind="text: getTitle()"></span></label>
    </div>
    <div class="payment-method-content">
        <!-- ko foreach: getRegion('messages') -->
        <!-- ko template: getTemplate() --><!-- /ko -->
        <!--/ko-->
        <div class="payment-method-billing-address">
            <!-- ko foreach: $parent.getRegion(getBillingAddressFormName()) -->
            <!-- ko template: getTemplate() --><!-- /ko -->
            <!--/ko-->
        </div>
        <div class="checkout-agreements-block">
            <!-- ko foreach: $parent.getRegion('before-place-order') -->
            <!-- ko template: getTemplate() --><!-- /ko -->
            <!--/ko-->
        </div>
        <div class="actions-toolbar">
            <div class="primary">
                <button class="action primary checkout"
                        type="submit"
                        data-bind="
                        click: placeOrder,
                        attr: {title: $t('Place Order')},
                        css: {disabled: !isPlaceOrderActionAllowed()},
                        enable: (getCode() == isChecked())
                        "
                        disabled>
                    <span data-bind="i18n: 'Place Order'"></span>
                </button>
            </div>
        </div>
    </div>
</div>
```



You maybe also interested in [Magento 2 Create Shipping Methods](https://www.mageplaza.com/magento-2-create-shipping-method/) to custom the shipping methods as expected.
