# Android SDK 0.7

## Getting started

### Account Setup

To ensure the authenticity of the interactions between your app and Truecaller, you need to supply us with your package name and SHA-1 signing-fingerprint.

Get the package name from your AndroidManifest.xml file. Use the following command to get the fingerprint:

```java
keytool -list -v -keystore mystore.keystore
```

Once we have received the package name and the SHA-1 signing-fingerprint, we will provide you with a unique "PartnerKey".

### Android Studio Setup

1. Ensure that your Minimum SDK is API 16: Android 4.1;
2. Add the provided truesdk-0.7.aar file into your libs folder. Example path: /app/libs/ ;
3. Open the build.gradle of your application module and firstly ensure that your lib folder can be used as a repository:

    ```java
    repositories {
        flatDir {
            dirs 'libs'
        }
    }
    ```
    
    Secondly add the compile dependency with the latest version of the TrueSDK aar:

    ```java
    dependencies {
        compile(name: "truesdk-0.6", ext: "aar")
    }
    ```

4. Open your strings.xml file. Example path: /app/src/main/res/values/strings.xml and add a new string with the name "partnerKey" and value as your "PartnerKey".
5. Open your AndroidManifest.xml and add a meta-data element to the application element:
 
    ```java
    <application android:label="@string/app_name" ...>
    ...
    <meta-data android:name="com.truecaller.android.sdk.PartnerKey" android:value="@string/partnerKey"/>
    ...
    </application>
    ```

6. Add the TrueButton view in the selected layout. You can have only one TrueButton per Activity

    ```java
    
    <com.truecaller.android.sdk.TrueButton
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    truesdk:truebutton_text="cont"/>
    ```

7. In your selected Activity

   - Either make your Activity implement ITrueCallback or create an instance. This interface has 3 methods: onSuccesProfileShared(TrueProfile), onFailureProfileShared(TrueError) and onOtpRequired()
   
   `
       private final ITrueCallback sdkCallback = new ITrueCallback() {
        @Override
        public void onSuccessProfileShared(@NonNull final TrueProfile trueProfile) {
            Toast.makeText(SignInActivity.this, "Verified without OTP! (Truecaller User): " + trueProfile.firstName,
                    Toast.LENGTH_SHORT).show();
        }

        @Override
        public void onFailureProfileShared(@NonNull final TrueError trueError) {
            Toast.makeText(SignInActivity.this, "onFailureProfileShared: " + trueError.getErrorType(), Toast
                    .LENGTH_SHORT).show();

        }

        @Override
        public void onOtpRequired() {
	    TrueSDK.getInstance().requestVerification("IN", phone, apiCallback);
        }
    };
   `
    
   - Similarly, make your Activity implement OtpCallback or create an instance. This interface has 2 methods: onOtpSuccess(int, Bundle) and onOtpFailure(int, TrueException)
   
   `
       static final OtpCallback apiCallback = new OtpCallback() {

        @Override
        public void onOtpSuccess(int requestCode, @Nullable Bundle bundle) {
            if (requestCode == OtpCallback.MODE_OTP_SENT) {
                Toast.makeText( mContext, "OTP Sent", Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(mContext, "Verified With OTP", Toast.LENGTH_SHORT).show();
                String accessToken = bundle.getString("TC_KEY_EXTRA_ACCESS_TOKEN");
            }
        }

        @Override
        public void onOtpFailure(final int requestCode, @NonNull final TrueException e) {
            Toast.makeText(mContext, "OnFailureApiCallback: " + e.getExceptionMessage(), Toast
                    .LENGTH_SHORT).show();
        }
    };
   `
    
    
   - Initialise the TrueSDK in the onCreate method:
    
     ```java
     TrueSDK.init(this, sdkCallback);
     ```
    
    (Optional) You can set an unique requestID for every profile request with     `TrueSDK.getInstance().setRequestNonce(customHash);`

     - Initialise the TrueButton in the onCreate method:

      ```java
      TrueButton trueButton = findViewById(R.id.com_truecaller_android_sdk_truebutton);
      ```
    
   - Add in the onActivityResult method the following condition:

      ```java
      TrueSDK.getInstance().onActivityResultObtained(resultCode, data);
      ```
      
   - Write all the relevant logic in onSuccesProfileShared(TrueProfile) for displaying the information you have just received and onFailureProfileShared(TrueError) for handling the error and notify the user. If truecaller app is not installed on the device, the control would be passed to onOtpRequired method. You can initiate the OTP verification flow from within this callback method by using :
   `
   TrueSDK.getInstance().requestVerification("IN", PHONE_NUMBER_STRING, OtpCallback);
   
   	Here, the first parameter is the country code of the mobile number on which the OTP needs to be trigerred
   	and PHONE_NUMBER_STRING should be the 10-digit mobile number of the user
   `

     The control would now pass to one of the method of OtpCallback. Write the relevant logic
     
  	 (Optional)  
     In order to use a custom button instead of the default TrueButton call trueButton.onClick(trueButton) in its onClick listner. Make sure your button follow our visual guidelines.

### Advanced and Optional

#### A. Server side Truecaller Profile authenticity check [ for users who verified via Truecaller app consent flow ]

Inside TrueProfile class there are 2 important fields, payload and signature. Payload is a Base64 encoding of the json object containing all profile info of the user. Signature contains the payload's signature. You can forward these fields back to your backend and verify the authenticity of the information by:

1. Fetch Truecaller public keys using this api: https://api4.truecaller.com/v1/key (you need to fetch the keys only if you have never done it or if you cannot verify the signature with any of the already cached keys);
2. Loop through the public keys and try to verify the signature and payload;

IMPORTANT: TrueSDK already verifies the authenticity of the response before forwarding it to the your app.

#### B. Request-Response correlation check

Every request sent via a Truecaller app that supports TrueSDK 0.6 has a unique identifier. This identifier is bundled into the response for assuring a correlation between a request and a response. If you want you can check this correlation yourself by:

1. Generate the request identifier via the TrueClient: `mTrueClient.generateRequestNonce();` or use your own Nonce set with `mTrueClient.setRequestNonce(customHash);`
2. `In ITrueCallback.onSuccesProfileShared(TrueProfile)` verify that the previously generated identifier matches the one in TrueProfile.requestNonce.

IMPORTANT: TrueSDK already verifies the Request-Response correlation before forwarding it to the your app.
