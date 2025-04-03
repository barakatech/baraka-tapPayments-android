# Why the need of this fork?

At baraka we allow users to input their debit card infos whenever they want to add a debit card as a deposit method. We already had a UI form where users can input their account holder name, card number, expiry and CVV. We didn't want to use Tap Payments UI form and keep our own. But we want also to use Tap Payments tokenization process. That's why we create this fork.

# Fork changes

In the source repository, when initializing the tap payments card view, it only allows us to inject card number and card expiry. We can't pass card holder name and CVV forcing us to use Tap Payments UI form.
In this fork, you will be able to pass card holder name and CVV directly from your UI.

Code changes:
- Synced with latest 1.0.4 changes
- Exclude `com.flurry.android` from `analytics` module to minimize binary size
- Add `cardCvv` and `cardHolderName` parameters into `configureWithTapCardDictionaryConfiguration` method inside `class TapCardConfiguration`
- Add `removeTapCardStatusDelegate` to remove tap card status delegate and avoid redundant API calls inside `class TapCardConfiguration`
- Add `cardExtraPrefillPair` to hold references for card holder name and Cvv into `class TapCardKit`
- Make `cardUrlPrefix` optional to nullify it once got either success or failure
- Add `cardCvv` and `cardHolderName` parameters into `init` method to initialize `cardExtraPrefillPair` inside `class TapCardKit`
- Check if `cardUrlPrefix` is null to get `cardUrlPrefix` avoiding redundant API calls
- Add `cardCvv` and `cardHolderName` parameters into `fillCardNumber` method to start tokenization process inside `class TapCardKit`
- Nullify `cardUrlPrefix` on success or failure delegate methods
- Add `removeTapCardStatusDelegate` to nullify `tapCardStatusDelegate` inside `object CardDataConfiguration`

# Integration Flow
1. Add Tap Payments dependency to your gradle file
``` groovy
com.github.barakatech:baraka-tapPayments-android:v1.1.19
```
2. Add Tap Payments Card Kit UI element to your XML layout and don't forget to hide it
  ```xml
  <company.tap.tapcardformkit.open.web_wrapper.TapCardKit
    android:id="@+id/tapCardForm"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toBottomOf="@id/field_cvv"
    app:layout_constraintEnd_toEndOf="parent"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:visibility="gone"/>
  ```
3. Get the debit card infos from your UI
  ```kotlin
  data class Card(
    val number: String,
    val expiryDate: String,
    val cvv: String,
    val holder: String
  )
  ```
4. Get Tap Payments parameters
   It should be a dictionnary of type `LinkedHashMap<String,Any>`.
   It should have the mandatory keys `scope`, `operator`, `purpose`, `order`, `customer` and `merchant`.
   Please refer to the source [documentation](https://developers.tap.company/docs/card-sdk-android#advanced-integration) to see examples.

```kotlin
data class TapPaymentsManagerImpl @Inject constructor(@ApplicationContext private val publicKey: String): TapPaymentsManager {
  private val _configuration: LinkedHashMap<String,Any>
    
  init {
    val operator = HashMap<String,Any>()
    operator["publicKey"] = publicKey

    val order = HashMap<String,Any>()
    order["id"] = ""
    order["amount"] = 1
    order["currency"] = "SAR"
    order["description"] = ""
    order["reference"] = ""

    val phone = java.util.HashMap<String,Any>()
    phone["countryCode"] = "+965"
    phone["number"] = "88888888"

    val contact = java.util.HashMap<String,Any>()
    contact["email"] = "tap@tap.company"
    contact["phone"] = phone

    val name = java.util.HashMap<String,Any>()
    name["lang"] = "en"
    name["first"] = "TAP"
    name["middle"] = ""
    name["last"] = "PAYMENTS"

    val customer = java.util.HashMap<String,Any>()
    customer["nameOnCard"] = "test"
    customer["contact"] = contact
    customer["name"] = listOf(name)

    _configuration = LinkedHashMap()
    _configuration["operator"] = operator
    _configuration["scope"] = "Token"
    _configuration["order"] = order
    _configuration["customer"] = customer
    _configuration["purpose"] = "Transaction"
    _configuration["merchant"] = //Your merchant id
  }

  fun getConfiguration(): LinkedHashMap<String, Any> {
    return _configuration
  }
}
```
5. Conform to Tap Payments delegate and handle success, error and invalid inputs
```kotlin
internal class AddCardFormFragment: TapCardStatusDelegate <- Conform your class with TapCardStatusDelegate
override fun onError(error: String) { ... }
override fun onSuccess(data: String) {
  TapCardConfiguration.removeTapCardStatusDelegate() <- Add this to avoid multiple API calls
  ...
}

override fun onValidInput(isValid: String) {
  val isValidInput = isValid.toBooleanStrict()
  if (isValidInput) {
    binding.tapCardForm.generateTapToken() <- binding.tapCardForm is a reference to UI element placed in your XML layout on step 1
  }
}
```
6. Call `initTapCardSDK` with debit card infos from step 3 and tap payments parameters from step 4
```kotlin
data class TapPaymentsInput(
    val form: Card,
    val configuration: LinkedHashMap<String,Any>
)

private fun configureTokenization(input: TapPaymentsInput) {
  val ctx = requireContext()

  TapCardConfiguration.configureWithTapCardDictionaryConfiguration(
    ctx,
    binding.tapCardForm,
    input.configuration,
    this@AddCardFormFragment, <- Here put the reference to your fragment
    cardNumber = input.form.number,
    cardExpiry = input.form.expiryDate,
    cardCvv = input.form.cvv,
    cardHolderName = input.form.holder
  )
}
```

## For more infos please refer to source [documentation](https://developers.tap.company/docs/card-sdk-android)

