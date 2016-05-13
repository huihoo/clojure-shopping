Clojure Shopping
=========

Online store backend/api written in Clojure

JSON api, supports paginated product list, authentication, user account, cart and payment.

Database is easily interchangeable thanks to data provider layer([com.novemberain/monger "1.7.0"], https://github.com/michaelklishin/monger). Currently MongoDB is used.

      (:require[monger.core :as mg]
               [monger.query :as mq]
               [monger.collection :as mc]
               [monger.operators :refer :all]
               [monger.result :refer [ok? has-error?]])

![mongodb](http://wiki.huihoo.com/images/f/f3/Clojushop-mongodb.png)

New version support Apache Cassandra. image storage use AWS S3.

Note that this is a learning project - use at your own risk.


##### Start server:
```
lein ring server-headless
```

###### Example curl requests:

Get products:

```
curl --request GET 'http://localhost:3000/products?st=0&sz=4&scsz=640x960'
```

Login (note use of cookie store - this will allow us to send next requests authenticated):

```
curl -i -H "Content-Type: application/json" -X POST -d '{"una":"user1", "upw":"test123"}' http://localhost:3000/user/login  --cookie "cookies.txt" --cookie-jar "cookies.txt" --location --verbose

curl -i -H "Content-Type: application/json" -X POST -d '{"una":"huihoo", "upw":"huihoo"}' http://localhost:3000/user/login  --cookie "cookies.txt" --cookie-jar "cookies.txt" --location --verbose
```


Get cart (authenticated request):

```
curl -i -H --request GET 'http://localhost:3000/cart?scsz=640x960'  --cookie "cookies.txt" --cookie-jar "cookies.txt" --location --verbose
```

##### Unit tests:
```
lein test clojushop.test.handler
```

The components can be tested separatedly:
```
lein test :only clojushop.test.handler/test-products
lein test :only clojushop.test.handler/test-cart
lein test :only clojushop.test.handler/test-users
lein test :only clojushop.test.handler/test-payment
```

##### Example client app (iOS):

https://github.com/i-schuetz/clojushop_client_ios


##### Api

Path  | Request type  | Authenticated  | Description  | Params
------------- | ------------- | ------------- | ------------- | -------------
/products  |  GET  | No |  Gets products list  | st: Page start, sz: Page size, scsz: Screen size e.g. 640x960
/products/add  | POST | Yes  | Adds a product |  na: name, des: description, img: image, pr: price, se: seller
/products/edit  | POST | Yes | Edits a product | na: name, des: description, img: image, pr: price, se: seller (all optional, but at least 1 required)
/products/remove  | POST | Yes | Removes a product | id: product id
/cart  | GET | Yes | Gets cart | scsz: Screen size e.g. 640x960
/cart/add  | POST | Yes | Adds a product to cart | id: product id
/cart/remove  | POST | Yes | Removes a product from cart | id: product id
/cart/quantity | POST | Yes | Sets the quantity of cart item (upsert) | id: product id, qt: quantity
/user | GET | Yes | Gets current user |
/user/register | POST | No | Registers a user | na: name, em: email, pw: password
/user/edit | POST | Yes | Edits current user | na: name, em: email, pw: password (all optional, but at least 1 required)
/user/login | POST | No | Logs in a user | username: name, password: password
/user/logout | GET | Yes | Logs out current user |
/user/remove | GET | Yes | Removes current user |
/pay | POST | Yes | Executes payment and empties cart | to: credit card token, v: amount c: currency ISO code


The unit tests in https://github.com/i-schuetz/clojushop/blob/master/test/clojushop/test/handler.clj do request calls and response processing like a normal client and can help with further understanding about how to use the api.

##### Status codes

All api responses have a status flag. Possible codes:

Code  | Meaning
------------- | -------------
0  | Unspecified error
1  | Success
2  | Wrong params
3  | Not found
4  | Validation error
5  | User already exists
6  | Login failed
7  | Not authenticated


Also, the results delivered by the dataprovider have status. Possible codes:

Code  | Meaning
------------- | -------------
0  | Unspecified error
1  | Success
2  | Bad id
3  | User already exists
4  | Not found
5  | Database internal error

The api and dataprovider status codes are independent of each other. The api handler (https://github.com/i-schuetz/clojushop/blob/master/src/clojushop/handler.clj) decides how to handle database result codes. Usually a map will be used to determine api code.


##### Image resolutions

When products are added to the database, the images have to be grouped in resolution categories. A resolution category is a simple numeric identifier. The meaning of the identifier is not fixed, and it depends where and how it's used.

Best explained with an example: Products are used in a list and a detailed view. List uses small images. Detail view uses big images. For testing we will use only 2 resolution categories - one will stand for "low" and the other for "high".

The img field of a product in the database would look like this (this implementation will be improved to avoid repeating the base path):

```
:img {
          :pl {
            :1 "http://ivanschuetz.com/img/cs/product_list/r1/blueberries.png"
            :2 "http://ivanschuetz.com/img/cs/product_list/r2/blueberries.png"}
          :pd {
            :1 "http://ivanschuetz.com/img/cs/product_details/r1/blueberries.png"
            :2 "http://ivanschuetz.com/img/cs/product_details/r2/blueberries.png"}
}
```

Here :pl stands for product-list and :pd for product-details. :1 is our low res category and :2 is high res (in a serious application we would use much more categories, e.g. specific for iPhone retina, Android (ranges), tablets, etc).

In total, we have 4 images. 2 use cases (list and details), and 2 available resolutions for each use case.

When the client makes a request to get items that contain images, it must send the screen size as "scsz" parameter. Example:
"scsz":"640x960"

In the function get-res-cat [screen-size] in our handler  (https://github.com/i-schuetz/clojushop/blob/master/src/clojushop/handler.clj), we map this screen size to a resolution category. The algorithm to do this can be anything - for demonstrative purposes, we use this:

    (if (< (Integer. width) 500) :1 :2)

This is, if the screen width is less than 500px we map this to resolution category 1 and if it's bigger to 2.

The items will then be filtered accordingly, such that the client gets only images suitable for their screensize. The image field of product in response would look like this:

```
"img":{"pd":"http://ivanschuetz.com/img/cs/product_details/r2/blueberries.png","pl":"http://ivanschuetz.com/img/cs/product_list/r2/blueberries.png"}
```

This is a very flexible implementation, since the client doesn't have to be modified when new resolutions are supported, and server can add arbitrary categories or new logic to determine which images fit best a certain screen size. Client only tells server e.g. "I want the images for the products list, and my screen size is 640x960", and uses whatever images the server delivers. This also works with orientation change, without additional changes - the client sends what currently is width and height and the server determines what fits best. It is not necessary to add additional identifiers for orientation or device type. Yet, orientation change opens the need for an improvement, namely that the client should not have to repeat the request only to get the images for the new orientation. This will probably be solved by calculating both resolutions categories interchanging width and height and send the client both images. In this case the request would keep the same, but the processing of the response would have to be adjusted.

##### Payment
* Alipay (支付宝)
* Weixin Payment (微信支付)
* Stripe

Stripe is used as payment system. A Stripe user account is necessary to test payments. The Stripe secret key has to be inserted in the calls. There is a placeholder in handler.clj called "your_stripe_secret_key" for this.

The correspoding public key has to be inserted in the client.

Currently the app suppors a basic credit card payment, using a credit card token. The client application gets the credit card data from the user, sends it to Stripe's api to get credit cart token, and then sends the token to Clojushop, together with the transaction amount and currency (using /pay service). Clojushop, then, calls the Stripe api with this data in order to do the transaction. The transactions show immediately in Stripe's dashboard.

Test credit card data:

4242424242424242 with any CVC and a valid expiration date.

More information about testing Stripe here: https://stripe.com/docs/testing

### Apache Cassandra Storage model and JSON schema
* Users keyspace
* Products keyspace
* Carts keyspace
* Orders keyspace
* Fiance keyspace
* Rules keyspace

##### Currency

In order to allow maximal flexibility, each product is saved with a currency. This may be subject to change, since it's difficult to handle the payment. Currently, the payment service accepts only one currency.

##### TODOs

Needs lots of online shop relevant stuff, like SKUs (currently Mongo id is used as identifier - very bad!), stock/inventory, improved security, validation, internationalization, etc. And of course, more features!

support new payment system:
* Alipay (支付宝)
* Weixin Payment (微信支付)

Integration Rails like framework: [Conjure](https://github.com/macourtney/Conjure)
* [ring](http://github.com/mmcgrana/ring)
* [clj-record](http://github.com/duelinmarkers/clj-record/) inspired by Ruby on Rails’ ActiveRecord
* [hiccup](https://github.com/weavejester/hiccup)

#### License
Copyright (C) 2016 Huihoo

Copyright (C) 2014 Ivan Schütz

Release under Apache License v2. See LICENSE.txt for the full license, at the top level of this repository.
