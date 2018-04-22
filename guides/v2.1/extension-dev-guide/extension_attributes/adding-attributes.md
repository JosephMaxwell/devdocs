---
layout: default
group: extension-dev-guide
subgroup: 99_Module Development
title: Adding extension attributes to entity
menu_title: Adding extension attributes to entity
menu_order: 20
version: 2.1
github_link: extension-dev-guide/extension_attributes/adding-attributes.md
---

# Details:

Let's say you are a developer that is tasked with loading some additional product details from a Product Management System. This data doesn't fit into existing product attributes as stored and displayed in tabular form. You could modify the core code (a bad idea) or you could try to serialize the data to store it in attributes (extra work in storing and retrieving).

Instead, Magento offers the use of extension attributes. Extension attributes don't affect existing API data structures. Rather, these are available in the `getExtensionAttributes()` method on classes that extend `\Magento\Framework\Model\AbstractExtensibleModel`. Extension attributes are specified in configuration XML (`extension_attributes.xml`), expressed through auto-generated interfaces and utilized in the modules that you write.

### Potential Use Cases for Extension Attributes
* Loading unique customer information from an ERP
* Storing synchronization data for ERP integration or a custom fraud detection implementation

### Things to understand:
* You are responsible for loading data into the extension attributes. ([example](https://github.com/magento/magento2/blob/2.2-develop/app/code/Magento/Catalog/Model/Category/Link/SaveHandler.php))
* You are responsible for persisting changes made to the extension attributes. ([example](https://github.com/magento/magento2/blob/2.2-develop/app/code/Magento/Catalog/Model/Category/Link/ReadHandler.php))

The return types for extension attributes can be categorized into two groups: scalar and object. This article focuses on object return types as more implementation is required. 

### Steps to utilize extension attributes\:
1. Create `etc/extension_attributes.xml`:
  1. Create an `extension_attributes` node for the class you want to extend.
  2. Create a `attribute` child inside `extension_attributes` with the interface that will be returned and the camel-case name of the attribute. For scalar return types, set the type to be the scalar type to be returned (like `type="int"`). You can also append `[]` to denote an array (like `type="int[]"`).
2. Build the interface (not needed for scalar return types).
3. Build an implementation for the interface (don't forget to make the association in `di.xml`). This is not needed for scalar return types.
4. Create plugins for all methods that load the entity from persistence. For a product entity, this would be:
  * [`\Magento\Catalog\Api\ProductRepositoryInterface::get`](https://github.com/magento/magento2/blob/2.2-develop/app/code/Magento/Catalog/Api/ProductRepositoryInterface.php)
  * [`\Magento\Catalog\Api\ProductRepositoryInterface::getList`](https://github.com/magento/magento2/blob/2.2-develop/app/code/Magento/Catalog/Api/ProductRepositoryInterface.php)


# Tutorial:

<div class="bs-callout bs-callout-info" id="other-component-types">
  <p>This article demonstrates how to create an extension attribute with a product entity, product Repository and the {% glossarytooltip 377dc0a3-b8a7-4dfa-808e-2de37e4c0029 %}web API{% endglossarytooltip %} example. </p>
</div>

### Step 1: create the XML configuration

The configuration is stored in `etc/extension_attributes.xml` in your module.

**Scalar Configuration:**
{% highlight xml %}
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="integer_type" type="int" />
        <attribute code="integer_array_type" type="int[]" />
    </extension_attributes>
</config>
{% endhighlight %}

**Object Configuration:**
{% highlight xml %}
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="product_shipment_date" type="YourCompany\YourModule\Api\Data\CustomDataInterface" />
    </extension_attributes>
</config>
{% endhighlight %}

### Step 2: build the interface

``` php?start_inline=1

namespace YourCompany\YourModule\Api\Data;

class CustomDataInterface
{
  public function getProductShipmentDate(): \DateTime;
  
  public function setProductShipmentDate(\DateTime $date);
}

```

### Step 3: build an implementation for the interface

The implementation for the interface is where the data will be stored in memory while it is being utilized. The implementation can be as detailed as necessary, but often is simple getters and setters providing a public interface to hidden properties.

### Step 4: create plugins to load the data

You are responsible for the loading and persistence for data in the extension attributes. The first step is to add the extension attribute information to the product when the product is loaded. There are multiple places that entity data is loaded, so careful attention is necessary.

**Example for ProductRepositoryInterface:**
``` php?start_inline=1
public function afterGet
(
    \Magento\Catalog\Api\ProductRepositoryInterface $subject,
    \Magento\Catalog\Api\Data\ProductInterface $entity
) {
    try {
        $shipmentDate = $this->productShipmentDateRepository->getByProductId($entity->getId);
    } catch (NoSuchEntityException $ex) {
        return $entity;
    }
    
    $extensionAttributes = $entity->getExtensionAttributes() ?: $this->productExtensionFactory->create();
    
    $customExtensionAttribute = $this->productShipmentDateExtensionFactory->create();
    $customExtensionAttribute->setProductShipmentDate($shipmentDate);
    
    $extensionAttributes->setProductShipmentDate($customExtensionAttribute);
    $entity->setExtensionAttributes($extensionAttributes);
    
    return $entity;
}
```

But if some entity doesn't have implementation to fetch extension attributes, we will always retrieve `null` and each time when we fetch extension atrributes we need to check if they are `null` - need to create them. To avoid such code duplication, we need to create `afterGet` plugin for our entity with extension attributes.

Let's assume the product entity doesn't have any implementation of extension attributes, so our plugin might looks like this:

``` php?start_inline=1

use Magento\Catalog\Api\Data\ProductExtensionInterface;
use Magento\Catalog\Api\Data\ProductInterface;
use Magento\Catalog\Api\Data\ProductExtensionFactory;

class ProductAttributesLoad
{
    /**
     * @var ProductExtensionFactory
     */
    private $extensionFactory;

    /**
     * @param ProductExtensionFactory $extensionFactory
     */
    public function __construct(ProductExtensionFactory $extensionFactory)
    {
        $this->extensionFactory = $extensionFactory;
    }

    /**
     * Loads product entity extension attributes
     *
     * @param ProductInterface $entity
     * @param ProductExtensionInterface|null $extension
     * @return ProductExtensionInterface
     */
    public function afterGetExtensionAttributes(
        ProductInterface $entity,
        ProductExtensionInterface $extension = null
    ) {
        if ($extension === null) {
            $extension = $this->extensionFactory->create();
        }

        return $extension;
    }
}

```

And now need to bind our plugin to `ProductInterface`:

{% highlight xml %}
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Catalog\Api\Data\ProductInterface">
        <plugin name="ProductExtensionAttributeOperations" type="Magento\Catalog\Plugin\ProductAttributesLoad"/>
    </type>
</config>
{% endhighlight %}

## Extension Attributes Configuration:



In first case we will get the next result:

{% highlight xml %}
<product>
    <id>1</id>
    <sku>some-sku</sku>
    <custom_attributes><!-- Custom Attributes Data --></custom_attributes>
    <extension_attributes>
        <first_custom_attribute>1</first_custom_attribute>
        <second_custom_attribute>2</second_custom_attribute>
    </extension_attributes>
</product>
{% endhighlight %}

In second one:
{% highlight xml %}
<product>
    <id>1</id>
    <sku>some-sku</sku>
    <custom_attributes><!-- Custom Attributes Data --></custom_attributes>
    <extension_attributes>
        <our_custom_data>
                <first_custom_attribute>1</first_custom_attribute>
                <second_custom_attribute>2</second_custom_attribute>
        </our_custom_data>
    </extension_attributes>
</product>
{% endhighlight %}

<a href="https://github.com/magento/magento2-samples/tree/master/sample-external-links">Sample module on github</a>

In order to get product or list of products by Magento API you need to do API request to appropriate service (Product Repository in our case).

In Response we got object with next structure:

### Product response:

{% highlight xml %}
<product>
    <id>1</id>
    <sku>some-sku</sku>
    <custom_attributes><!-- Custom Attributes Data --></custom_attributes>
    <extension_attributes><!-- Here should we add extension attributes data --></extension_attributes>
</product>
{% endhighlight %}

### Product list response:

{% highlight xml %}
<products>
    <item>
        <id>1</id>
        <sku>some-sku</sku>
        <custom_attributes><!-- Custom Attributes Data --></custom_attributes>
        <extension_attributes><!-- Here should we add extension attributes data --></extension_attributes>
    </item>
    <item>
        <id>2</id>
        <sku>some-sku-2</sku>
        <custom_attributes><!-- Custom Attributes Data --></custom_attributes>
        <extension_attributes><!-- Here should we add extension attributes data --></extension_attributes>
    </item>
</products>
{% endhighlight %}
