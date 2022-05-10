  - [Authorization](#authorization)
  - [Statuses](#statuses)
    - [IPN](#IPN)
    - [Deposit](#deposit)
    - [Withdraw](#withdraw)
  - [IPN Instant Payment Notification](#ipn-instant-payment-notification)
    - [Creation](#creation)
    - [CallBack](#callback)
    - [Get Info About Invoice](#get-info-about-invoice)
    - [Get Invoices History](#get-invoices-history)
  - [Payments](#payments)
    - [History by Latest](#history-by-latest)
    - [History by Period](#history-by-period)


## Authorization



Follow 5 simple steps to use private methods:

* Before authorization, you need to log in to https://coinsbit.io/login or register if the user has not yet been created https://coinsbit.io/register.
* After authorization, you need to go to https://coinsbit.io/user/security-main and activate 2-factor authentication.
* After enabling 2-factor authentication, you need to go to the API manager to create API keys that will be required for authorization in the API https://coinsbit.io/user/api.
* To create keys, click Add Key and the 1st package of keys for connecting the API will be created. Key packages can be up to 5 pieces.
* Before using the apkey, they must be activated - for this, on the created package, click on the switch which will switch them to the "Active" state.


Each package will contain 3 keys:
apiKey - public key
apiSecret - private key
weKey - key for websockets




**Important!**

  `nonce` - used as a parameter to protect against DDoS attacks and an excessive number of API requests. This parameter is most often used through a timestamp with a minimum step per second. Each next request must be greater than the next one in the `nonce` parameters.



Let's look at a few examples of authorization in different development languages:

**PHP**
  <details  open>
  <summary>
  </summary>
  
  
```php
use GuzzleHttp\Client;

public function callApi()
{
    $apiKey = '';
    $apiSecret = '';
		$request = ''; // /api/v1/order/new
    $baseUrl = 'https://api.coinsbit.io';

    $data = [
        'request' => $request,
        'nonce' => (string)\Carbon\Carbon::now()->timestamp,
    ];

    $completeUrl = $baseUrl . $request;
    $dataJsonStr = json_encode($data, JSON_UNESCAPED_SLASHES);
    $payload = base64_encode($dataJsonStr);
    $signature = hash_hmac('sha512', $payload, $apiSecret);

    $client = new Client();
    try {
        $res = $client->post($completeUrl, [
            'headers' => [
                'Content-type' => 'application/json',
                'X-TXC-APIKEY' => $apiKey,
                'X-TXC-PAYLOAD' => $payload,
                'X-TXC-SIGNATURE' => $signature
            ],
            'body' => json_encode($data, JSON_UNESCAPED_SLASHES)
        ]);
    } catch (\Exception $e) {
        return response()->json(['error' => $e->getMessage()]);
    }

    return response()->json(['result' => json_decode($res->getBody()->getContents())]);
}
```

  </details>


**Nest.js or Node.js**
  <details  open>
  <summary>
  </summary>
  
  
```php

import { HttpService, Injectable } from '@nestjs/common';
import * as crypto from 'crypto';
import { AxiosResponse } from 'axios';

export interface InvoicePayload {
  success: boolean;
  message: string;
  result: {
    invoice: string;
    redirect_link: string;
    ticker: string;
    payment_method: string | null;
    invoiceAmount: string;
    success_url: string | null;
    error_url: string | null;
    order_number: string;
    callback_url: string | null;
  };
  code: number;
}

@Injectable()
export class PaymentService {
  constructor(private http: HttpService) {}
  createInvoice(): Promise<AxiosResponse<InvoicePayload>> {
    const now = Date.now();
    const input = {
      callback_url: 'https://callback.url',
      success_url: 'https://google.com/',
      error_url: 'https://google.com/',
      invoiceAmount: '100',
      order_number: 'test',
      request: '/api/v1/merchant/ipn_create',
      ticker: 'ETH',
      nonce: now,
    };
    const secret = '';
    const apiKey = '';
    const baseUrl = 'https://api.coinsbit.io';
    const payload = JSON.stringify(input, null, 0);
    console.log(payload);
    const jsonPayload = Buffer.from(payload).toString('base64');
    const encrypted = crypto.createHmac('sha512', secret).update(jsonPayload).digest('hex');
    return this.http
      .post<InvoicePayload>(`${baseUrl}/api/v1/merchant/ipn_create`, input, {
        headers: {
          'Content-type': 'application/json',
          'X-TXC-APIKEY': apiKey,
          'X-TXC-PAYLOAD': jsonPayload,
          'X-TXC-SIGNATURE': encrypted,
        },
      })
      .toPromise();
  }
}

```
  </details>

## Statuses

Statuses are used to determine the status of a payment at the current moment and have common names in payment systems

### IPN


* `INITIALIZATION` - initial status when created
* `PENDING` - waiting for payment from the user
* `SUCCESS` - successful payment
* `SUCCESS_WITH_SMALL_AMOUNT` - successful payment, but with user paid amount less than `invoiceAmount`
* `CANCEL` - unsuccessful invoice


### Deposit

* `PENDING` - waiting for payment 
* `SUCCESSFUL` - successful payment
* `IN PROCESS` - waiting for network confirmation
* `CANCELED` - unsuccessful invoice
* `UNKNOWN` - payment is unauthorized 

### Withdraw

* `PENDING` - waiting for success response from blockchain network
* `UNCONFIRMED_BY_ADMIN` - waiting for checking by admin in case if something went wrong
* `SUCCESSFUL` - withdrawal successful
* `CANCEL` - withdrawal unsuccessful



## IPN Instant Payment Notification 

To create an invoice for replenishment, each time you apply, you create your own details for the replenishment by the user, which are active 3 hours after creation.

If within 3 hours the account is not replenished according to the specified details on the invoice, the invoice is closed as `Cancel`.

CalBask will be activated only if the `callback_url` parameter was specified when creating an invoice - if the parameter was not specified, then information on a specific invoice can be obtained only through the `/api/v1/merchant/invoice_status` method.

To receive information on an invoice using `POST`, you need to call the `/api/v1/merchant/invoice_status` method with the `invoice` parameter of the payment information about which you want to receive.


To obtain historical data, use the `/api/v1/merchant/invoice_list` method with the specified from and to time required to obtain historical data.

Amount logic description 

* `Amount` - that is parameters which shows how exactly was credited to Coinsbit balance ( fee included ) 
* `ActualAmount` - that is parameters which shows how exactly was paid by your invoice or deposit ( without fee ) 
* `InvoiceAmount` - that is parameters which shows how mach need to pay by created invoice or transaction
* `Fee` - that is parameters which shows how mach need was charged by transaction ( if it was )




## Creation
  <details open>
  <summary>
  </summary>



```
/api/v1/merchant/ipn_create
```

Type `POST`


**Request Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
ticker | STRING | YES | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
payment_method | STRING | NO | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
success_url | STRING | YES | URL for redirecting user in success case of payment
error_url | STRING | YES | URL for redirecting user in unsuccessful case of payment
invoiceAmount | STRING | YES | Expected amount of replenishment
order_number | STRING | NO | Custom field for identify internal platform payments
callback_url | STRING | NO | URL for receiving CallBack of invoice status
request | STRING | YES | Link for API method
nonce | STRING | YES | Timestamp equal parameter for request quantity regulation (1 per second)

**Request:**


```javascript
{
      "ticker":"USDT",
      "payment_method":"TRX", 
      "success_url":"http://yoursite.com/success",
      "error_url":"http://yoursite.com/error",
      "callback_url":"https://yoursite.com/envirement",
      "invoiceAmount":"1000",
      "order_number":"ID_10",
      "request":"/api/v1/merchant/ipn_create",
      "nonce":"microtime(true)*1000",
}

```

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
redirect_link | STRING | URL to page of payment 
callback_url | STRING  | URL for receiving CallBack of invoice status
ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
invoiceAmount | STRING  | Expected amount of replenishment
success_url | STRING  | URL for redirecting user in successful case of payment
error_url | STRING  | URL for redirecting user in unsuccessful case of payment
order_number | STRING  | Custom field for identify internal platform payments



**Response:**
```javascript
{
    "success":"1"
    "message"
    "result":"Array"
        {
            "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
            "redirect_link":"https://coinsbit.io/merchant/9192b8ab-8582-4704-a48c-bd2cfcd08018",
            "callback_url":"https://yoursite.com//werwer",
            "ticker":"USDT",
            "payment_method:"ERC",
            "invoiceAmount":"1000",
            "success_url":"http://buy.ru",
            "error_url":"http://buy.ru",
            "order_number":"ID_10",
            "address":"0x21Dd5c13925407e5bCec3f27aB11a355a9Dafbe3",
            "status":"PENDING",
        }

    "code":"200"
}
```

</details>


### CallBack


If the `callback_url` parameter is specified in the request `/api/v1/merchant/generate_nolimit_invoice`, then when the invoice status changes, a `POST` request with invoice data will be sent to the specified URL `1 per min within an hour` after the invoice status is changed to `SUCCESSFUL` or `CANCEL`

  <details open>
  <summary>
  </summary>

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
invoiceAmount | STRING  | Expected amount of replenishment
actualAmount | STRING  | User paid amount
amount | STRING  | Credited to your balance in Coinsbit amount (fee deducted)
fee | STRING  | Amount which was deducted by receiving transaction by the Coinsbit (if exist)
sysAddress | STRING | Basic address which used by payment
address | STRING  | That is memo address which used by payments in Blockchain networks like Ripple, EOS etc. / Can be empty
hash | STRING  | ID of user payment in Blockchain network
type | STRING  | FIAT or CRYPTO - type of payment coin or currency
status | STRING  | Status of payment - read [Statuses](#statuses)
order_number | STRING  | Custom field for identify internal platform payments
createdAt | STRING |  Time of invoice creation in format('Y-m-d H:i:s')
updatedAt | STRING |  Time of invoice status updating in format('Y-m-d H:i:s')




**Response:**
```javascript
{
   "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
   "redirect_link":"https://coinsbit.io/merchant/9192b8ab-8582-4704-a48c-bd2cfcd08018",
   "ticker":"USDT",
   "payment_method":"ERC",
   "invoiceAmount":"1000",
   "actualAmount":"1100",
   "amount":"1078",
   "fee":"22",
   "sysAddress":"0xf353E0AB416b060da8D3E397679922EAcE7D78Db",
   "address":"10392923",
   "hash":"0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
   "type":"CRYPTO", 
   "status":"SUCCESS",
   "order_number":"ID_10",
   "createdAt":"2019-10-11 21:12:23",
   "updateAt":"2019-10-11 21:15:23",
}
```


  </details>




## Get Info About Invoice


  <details open>
  <summary>
  </summary>



```
/api/v1/merchant/invoice_status
```

Type `POST`


**Request Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
invoice| STRING | YES | Platform Internal invoice ID which was generated after invoice creation
request | STRING | YES | Link for API method
nonce | STRING | YES | Timestamp equal parameter for request quantity regulation (1 per second)


**Request:**


```javascript
{
   "invoice":"a3ad686e-66ce-420f-a360-c0d3e96d6885",
   "request":"/api/v1/merchant/invoice_status",
   "nonce":"microtime(true)*1000",
}

```

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
order_number | STRING  | Custom field for identify internal platform payments
invoiceAmount | STRING | YES | Expected amount of replenishment
ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
amount | STRING  | Credited to your balance in Coinsbit amount (fee deducted)
actualAmount | STRING  | User paid amount
fee | STRING  |  Amount which was deducted by receiving transaction by the Coinsbit (if exist)
status | STRING  |Status of payment - read [Statuses](#statuses)
address | STRING  | Address in Blockchain network for payment
txHash | STRING  | Transaction ID in Blockchain network
explorerLink | STRING  | URL to payment in Blockchain
createdAt | STRING  |  Time of invoice creation in format('Y-m-d H:i:s')
updatedAt | STRING |  Time of invoice status updating in format('Y-m-d H:i:s')




**Response:**
```javascript
{
   "success": true,
   "message": "",
   "result": 
      {

            "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
            "order_number":"ID_10"
            "invoiceAmount":"1000",
            "ticker":"USDT",
            "payment_method":"ERC",
            "currency":"Tether USD",
            "amount":"1078.00000000",
            "actualAmount":"1100.00000000",
            "fee":"22.000000000000000000",
            "status":"successful",
            "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
            "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
            "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
            "createdAt": "2019-10-11 21:12:23",
            "updatedAt": "2019-10-11 21:15:23",

      }
   "code": 200
}

```

</details>


## Get Invoices History


  <details open>
  <summary>
  </summary>



```
/api/v1/merchant/invoice_list
```

Type `POST`


**Request Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
from | TimeStamp | YES | Timestamp period from 
to | TimeStamp | YES | Timestamp period from 
request | STRING | YES | Link for API method
nonce | STRING | YES | Timestamp equal parameter for request quantity regulation (1 per second)


**Request:**


```javascript
{
   "from":"1617756100",
   "to":"1618756228",
   "nonce":"microtime(true)*1000",
   "request":"/api/v1/merchant/invoice_list",
}

```

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
order_number | STRING  | Custom field for identify internal platform payments
invoiceAmount | STRING | YES | Expected amount of replenishment
ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
amount | STRING  | Credited to your balance in Coinsbit amount (fee deducted)
actualAmount | STRING  | User paid amount
fee | STRING  |  Amount which was deducted by receiving transaction by the Coinsbit (if exist)
status | STRING  |Status of payment - read [Statuses](#statuses)
address | STRING  | Address in Blockchain network for payment
txHash | STRING  | Transaction ID in Blockchain network
explorerLink | STRING  | URL to payment in Blockchain
createdAt | STRING  |  Time of invoice creation in format('Y-m-d H:i:s'),
updatedAt | STRING |  Time of invoice status updating in format('Y-m-d H:i:s')




**Response:**
```javascript
{
   "success": true,
   "message": "",
   "result": 
      {
         "records": 
            [
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "invoiceAmount":"1000",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
                  "updatedAt": "2019-10-11 21:12:23",
               },
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "invoiceAmount":"1000",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
                  "updatedAt": "2019-10-11 21:12:23",
               }
            ]
      }
   "code": 200
}

```

</details>



## Payments



## History by Latest


  <details open>
  <summary>
  </summary>



```
/api/v1/payment/transactionslist
```

Type `POST`


**Request Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
ticker| STRING | YES | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
limit | STRING | YES | min:1 max:1000
offset | STRING | YES | Pagination, max:100
type | STRING | YES | `deposit` or `withdraw` - type of transaction (invoice = deposit=ipn)
request | STRING | YES | Link for API method
nonce | STRING | YES | Timestamp equal parameter for request quantity regulation (1 per second)


**Request:**


```javascript
{
   "ticker":"USDT",  
   "limit":"5",    //лимит записей которые вернет запрос min:1|max:1000
   "offset":"0",
   "type":"deposit",
   "request":"/api/v1/payment/transactionslist",
   "nonce":"microtime(true)*1000",

}

```

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
order_number | STRING  | Custom field for identify internal platform payments
txId | STRING  | Custom field for identify internal platform payments not equal `invoice`
Ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
Currency | STRING  | Currency name to payment ( Teather USD / Bitcoin / Litecoin) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
amount | STRING  | Credited to your balance in Coinsbit amount (fee deducted)
actualAmount | STRING  | User paid amount
fee | STRING  |  Amount which was deducted by receiving transaction by the Coinsbit (if exist)
status | STRING  |Status of payment - read [Statuses](#statuses)
address | STRING  | Address in Blockchain network for payment
txHash | STRING  | Transaction ID in Blockchain network
explorerLink | STRING  | URL to payment in Blockchain
createdAt | STRING  |  Time of invoice creation in format('Y-m-d H:i:s'),




**Response:**
```javascript
{
   "success": true,
   "message": "",
   "result": 
      {
         "records": 
            [
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "txid":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
               },
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "txid":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
               }
            ]
      }
   "code": 200
}

```

</details>


## History by Period


  <details open>
  <summary>
  </summary>



```
/api/v1/payment/transactionstimerangelist
```

Type `POST`


**Request Parameters:**

Name | Type | Mandatory | Description
------------ | ------------ | ------------ | ------------
from | TimeStamp | YES | Timestamp period from 
to | TimeStamp | YES | Timestamp period from 
type | STRING | YES | `deposit` or `withdraw` - type of transaction (invoice = deposit=ipn)
request | STRING | YES | Link for API method
nonce | STRING | YES | Timestamp equal parameter for request quantity regulation (1 per second)


**Request:**


```javascript
{
   "from":"1617756100",
   "to":"1618756228",
   "type":"deposit",
   "request":"/api/v1/payment/transactionstimerangelist",
   "nonce":"microtime(true)*1000",
}

```

**Response Parameters:**

Name | Type  | Description
------------ |  ------------ | ------------
invoice | STRING  | Platform Internal invoice ID
order_number | STRING  | Custom field for identify internal platform payments
txId | STRING  | Custom field for identify internal platform payments not equal `invoice`
ticker | STRING  | Currency ticker to payment ( USDT / USD / EUR / BTC ) 
Currency | STRING  | Currency name to payment ( Teather USD / Bitcoin / Litecoin) 
payment_method | STRING  | Default for coins - basic, Default for tokens - ERC, Can be for tokens: BSC, TRX
amount | STRING  | Credited to your balance in Coinsbit amount (fee deducted)
actualAmount | STRING  | User paid amount
fee | STRING  |  Amount which was deducted by receiving transaction by the Coinsbit (if exist)
status | STRING  |Status of payment - read [Statuses](#statuses)
address | STRING  | Address in Blockchain network for payment
txHash | STRING  | Transaction ID in Blockchain network
explorerLink | STRING  | URL to payment in Blockchain
createdAt | STRING  |  Time of invoice creation in format('Y-m-d H:i:s'),




**Response:**
```javascript
{
   "success": true,
   "message": "",
   "result": 
      {
         "records": 
            [
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "txid":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
               },
               {
                  "invoice":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "order_number":"ID_10"
                  "txid":"9192b8ab-8582-4704-a48c-bd2cfcd08018",
                  "ticker":"USDT",
                  "payment_method":"ERC",
                  "currency":"Tether USD",
                  "amount":"1078.00000000",
                  "actualAmount":"1100.00000000",
                  "fee":"22.000000000000000000",
                  "status":"successful",
                  "address": "0xf353E0AB416b060da8D3E397679922EAcE7D78Db"
                  "txHash": "0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "explorerLink": "https://etherscan.io/tx/0x67a4d966b5cf0a30d0b5223d5d55383ff69f4f22a64d109f18ac4f6e3f9d1a5b",
                  "createdAt": "2019-10-11 21:12:23",
               }
            ]
      }
   "code": 200
}

```

</details>