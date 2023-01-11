## 1. Project configuration

#### In `build.gradle` fiLe on project module level:

- set the min Android Sdk to 21: `minSdkVersion=21` 
- add dependency of the SDK:
```gradle
dependencies {
    implementation "pl.payone:android-sdk-sibs-paywall:$sibs_paywall_ver"
}
```
<br/>

#### Specity the repository for Sibs SDK dependency

###### NOTE!
> `authToken` can be obtained by sending a request to [sdk_sibs@payone.pl](mailto:sdk_sibs@payone.pl)

Depending on the project setup:
If the repositories resolution is specified in the project level `build.gradle` file:
```gradle
allprojects {
    repositories {
        maven {
            url "https://jitpack.io"
            credentials { username authToken }
        }
    }
}
```
Or if it is specified in the `settings.gradle` file:
```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        maven {
            url "https://jitpack.io"
            credentials { username authToken }
        }
    }
}
```
<br/>

#### AndroidManifest definitions

a) Add internet permission
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

b) Add TransactionActivity definition

TransactionActivity extends on the `AppCompatActivity`. Therefore `Theme.AppCompat.*` styles should be used.
Using `Theme.AppCompat.*.NoActionBar` family styles will hide the ActionBar

By default when the WebView used internally by SDK gets reloaded on every activity recreation (f.eks. rotation change) 
It is recommended to block this behavior by adding the following parameter to the activity definition


```xml
android:configChanges="orientation|keyboard|keyboardHidden|screenSize"
```

Sibs environment configurations are provided by Sibs and passed as metadata:
Example `TransactionActivity` definition:
```xml
<activity
    android:name="com.sibs.sdk.TransactionActivity"
    android:theme="@style/Theme.TransactionActivity" 
    <!--SDK actionBar title-->
    android:label="SDK"> 
    <meta-data
        android:name="sibs_access_token"
        android:value="my_sibs_access_token" />
    <meta-data
        android:name="sibs_client_id"
        android:value="my_sibs_client_id" />
    <meta-data
        android:name="sibs_api_url"
        android:value="my_sibs_api_url" />
    <meta-data
        android:name="sibs_web_url"
        android:value="my_sibs_web_url" />
</activity>
```
<br/>

## 2. Starting the Sibs SDK

#### Create transaction parameters
In order to call the transaction, the following parameters must be set using the builder for `TransactionParams`:
```kotlin
val transactionParamsBuilder = TransactionParams.Builder()

    //required parameters
    .terminalId(terminalId) //not negative
    .transactionDescription(transactionDescription) //not blank
    .transactionId(transactionId) //not blank, limited to 35 alphanummeric characters
    .amount(amount) //not negative
    .currency(currency) //3 characters expected
    .paymentMethods(paymentMethods) //not empty

    //optional parameters
    .merchantTransactionDescription(merchantTransactionDescription)
    .shopUrl(shopUrl)
    .client(client)
    .email(email)
    .billingAddress(billingAddress)
    .shippingAddress(shippingAddress)
```

If provided parameters don't meet the assertions the 'IllegalArgumentException` exception will be thrown: 
```
    val transactionParams = runCatching { transactionParamsBuilder.build() }
        .onFailure { Log.w(TAG, "Couldn't build the transaction params: $[it]") }
        .getOrNull()
```

Optionally it is possible to append `AddressParams` object to the `TransactionParams` builder:
```
    val addressBuilder = AddressParams.Builder()
            
            //required parameters
            .street1(street)
            .city(cite)
            .zip(zip)
            .country(PL)

            //optional parameters
            .street2(street2)

    val shippingAddress = runCatching { addressBuilder.build() }
            .onFailure { Log.w((TAG, "Shipping address params $[it]") }
            .getOrNull()
    val billingAddress = runCatching { addressBuilder.build() }
            .onFailure { Log.w(TAG, "Billing address params $[it]") }
            .getOrNull()
        
    transactionParamsBuilder
            .billingAddress(billingAddress)
            .shippingAddress(shippingAddress)

```
<br/>

#### Start the payment flow:
```
    val sibsSdkIntent = TransactionActivity.getIntent(this, transactionParams)
    sdkActivityLauncher.launch(sibsSdkIntent)
```
<br/>

## 2. Consume the Sibs SDK result:

Result of the transaction can be obtained from activity result intent:
```
val transferResult = resultIntent.let(TransactionActivity::parseResult)
```

Example of successful transaction handling:
```
    private val sdkActivityLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { activityResult ->
        if (Activity.RESULT_OK == activityResult.resultCode) {
            val transferResult = activityResult.data
                ?.let(TransactionActivity::parseResult)
                ?: return@registerForActivityResult

            if (transferResult.isSuccess) {
                Log.d(TAG, transferResult.transactionId!!)
            } 
        } 
    }
```
<br/>

## 3. Optional transaction status check:

It is possible to get the detailed transaction status data and compare the result with the documentation provided by Sibs
```
    val statusCheckService = SibsPaymentStatusService(context)
    lifecycleScope.launch { 
        val statusCheckResponse = statusCheckService.check(transactionId)
        if (statusCheckResponse != null) {
            Log.d(TAG, response::class.java.simpleName)
        } 
    }
```
<br/>

## 4. Errors handling
The Sibs SDK can return errors if an operation couldn't be fulfilled.

#### Transaction errors:
Defined as subclasses of the `SibsSdkError` sealed class\
`WebViewNotAvailable` - WebView is not available on this device or is being updated\
`MissingTransactionParams` - Transaction Params not passed to `TransactionActivity`, please use the TransactionActivity.getIntent for intent creation\
`SDKNotConfigured` - Sibs application or Sibs environment metadata are missing\
`CheckoutError` - Couldn't perform the checkout. Please refer to the Checkout documentation: [https://developer.sibsapimarket.com/sibs-qly/sibslabs/api/checkout-sandbox](https://developer.sibsapimarket.com/sibs-qly/sibslabs/api/checkout-sandbox)

#### Payment status check errors:
Defined as subclasses of the `PaymentStatusCheckResponse` sealed class\
`SDKNotConfigured` - Sibs application or Sibs environment metadata are missing\
`Error` - Couldn't perform transaction status check. Please refer to the Payment Status documentation [https://developer.sibsapimarket.com/sibs-qly/sibslabs/api/checkout-sandbox](https://developer.sibsapimarket.com/sibs-qly/sibslabs/api/checkout-sandbox)
