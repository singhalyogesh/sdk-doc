# Android SDK 0.7

## Getting started

### Account Setup

To ensure the authenticity of the interactions between your app and Truecaller, you need to supply us with your package name and SHA-1 signing-fingerprint.

Get the package name from your AndroidManifest.xml file. Use the following command to get the fingerprint:

```java
keytool -list -v -keystore mystore.keystore
```

Once we have received the package name and the SHA-1 signing-fingerprint, we will provide you with a unique "PartnerKey".

### Using the SDK with your Android Studio Project

1. Ensure that your Minimum SDK is API 16: Android 4.1
2. Add the provided truesdk-0.7-releasePartner.aar file into your libs folder. Example path: /app/libs/ 
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
        compile(name: "truesdk-0.7-releasePartner", ext: "aar")
    }
    ```

4. Open your strings.xml file. Example path: /app/src/main/res/values/strings.xml and add a new string with the name "partnerKey" and value as your "PartnerKey"

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

7. Initialise the TrueSDK in the onCreate method:
    
     ```java
     TrueSDK.init(this, sdkCallback);
     ```
    
    (Optional) You can set an unique requestID for every profile request with     	`TrueSDK.getInstance().setRequestNonce(customHash);`

 8. Initialise the TrueButton in the onCreate method:

      ```java
      TrueButton trueButton = findViewById(R.id.com_truecaller_android_sdk_truebutton);
      ```
    
 9. Add the following condition in the onActivityResult method:

      ```java
      TrueSDK.getInstance().onActivityResultObtained(resultCode, data);
      ```
      
 10. In your selected Activity

   - Either make your Activity implement ITrueCallback or create an instance. This interface has 3 methods: onSuccesProfileShared(TrueProfile), onFailureProfileShared(TrueError) and onOtpRequired()
   
   ```
       private final ITrueCallback sdkCallback = new ITrueCallback() {
        @Override
        public void onSuccessProfileShared(@NonNull final TrueProfile trueProfile) {
	// This method is invoked when either the truecaller app is installed on the device and the user gives his consent 	   // to share his truecaller profile OR when the user has already been verified before on the same device using the 	     //	same number and hence does not need OTP to verify himself again. 
	
            Toast.makeText(SignInActivity.this, "Verified without OTP! (Truecaller User): " + trueProfile.firstName,
                    Toast.LENGTH_SHORT).show();
        }

        @Override
        public void onFailureProfileShared(@NonNull final TrueError trueError) {
	// This method is invoked when some error occurs or if an invalid request for verification is made. 
	
            Toast.makeText(SignInActivity.this, "onFailureProfileShared: " + trueError.getErrorType(), Toast
                    .LENGTH_SHORT).show();

        }

        @Override
        public void onOtpRequired() {
	// This method is invoked when truecaller app is not present on the device or if the user does not want to share his 	     // truecaller profile and hence, OTP verification is required to complete the verification flow
	// You can initiate the OTP verification flow from within this callback method by using :
	   
	    TrueSDK.getInstance().requestVerification("IN", phone, apiCallback);
	    
	    
   	//  Here, the first parameter is the country code of the mobile number on which the OTP needs to be trigerred
	//  and PHONE_NUMBER_STRING should be the 10-digit mobile number of the user
        }
    };
    
   ```
    
   Write all the relevant logic in onSuccesProfileShared(TrueProfile) for displaying the information you have just received and onFailureProfileShared(TrueError) for handling the error and notify the user.
   
   Similarly, make your Activity implement OtpCallback or create an instance ( Once the OTP verification is triggered using the 'requestVerification' method, the control would then be passed to OtpCallback ) .
   This interface has 2 methods: onOtpSuccess(int, Bundle) and onOtpFailure(int, TrueException)
   
   ```
       static final OtpCallback apiCallback = new OtpCallback() {

        @Override
        public void onOtpSuccess(int requestCode, @Nullable Bundle bundle) {
            if (requestCode == OtpCallback.MODE_OTP_SENT) {
	    // This method is invoked when the OTP has been sent to the input mobile number.
	    // You can now ask the user to input the 6-digit OTP code sent to him via SMS and ask for his first name and      		  // last name
	    
                Toast.makeText( mContext, "OTP Sent", Toast.LENGTH_SHORT).show();
            } else {
	    // This method is invoked when the user has successfully input the correct OTP code along with his first name               // and last name and is verified successfully by the SDK
	    
                Toast.makeText(mContext, "Verified With OTP", Toast.LENGTH_SHORT).show();
            }
        }

        @Override
        public void onOtpFailure(final int requestCode, @NonNull final TrueException e) {
	// Invoked when some error has occured while verifying the provided mobile number via OTP
            Toast.makeText(mContext, "OnFailureApiCallback: " + e.getExceptionMessage(), Toast
                    .LENGTH_SHORT).show();
        }
    };
    
   ```
         
   To complete the verification once the OTP has been sent to the provided mobile number, complete the verification process by calling the following method from within your activity :
   
   ```
   TrueProfile profile = new TrueProfile.Builder(firstName, lastName).build();
   TrueSDK.getInstance().verifyOtp(profile, otp, apiCallback);
   
   // 'getInstance' method returns the control to the OtpCallback as defined in the section above
   ```

  (Optional)  
  In order to use a custom button instead of the default TrueButton call trueButton.onClick(trueButton) in its onClick listner. Make sure your button follow our visual guidelines.

### Advanced and Optional

#### A. Server side Truecaller Profile authenticity check [ for users who verified via Truecaller app consent flow ]

Inside TrueProfile class there are 2 important fields, payload and signature. Payload is a Base64 encoding of the json object containing all profile info of the user. Signature contains the payload's signature. You can forward these fields back to your backend and verify the authenticity of the information by:

1. Fetch Truecaller public keys using this api: https://api4.truecaller.com/v1/key (you need to fetch the keys only if you have never done it or if you cannot verify the signature with any of the already cached keys);
2. Loop through the public keys and try to verify the signature and payload;

IMPORTANT: Truecaller SDK already verifies the authenticity of the response before forwarding it to the your app.

#### B. Server side Truecaller Profile authenticity check [ for users who verified via Truecaller OTP flow ]

In OnSuccess method of OtpCallback, in case of MODE_VERIFIED, you can fetch the access token string. You can forward this field back to your backend and verify the authenticity of the information by using the following endpoint:

**Endpoint:**  
https://api4.truecaller.com/v1/otp/installation/validate/{accessToken}

**Method:**  
POST

**Header parameters:**

| **Parameter [Type]** | **Required** | **Description**         | **Example**                            |
| -------------------  | ------------ | ----------------------- | -------------------------------------- |
| appKey [String]      | yes          | Applications secret key | zHTqS70ca9d3e988946f19a65a01dRR5e56460 |

**Request path parameter:**

| **Parameter [Type]** | **Required** | **Description**                                                                   | **Example**                            |
| -------------------  | ------------ | --------------------------------------------------------------------------------- | -------------------------------------- |
| accessToken          | yes          | token granted for the partner for the respective user number that initiated login | "71d8367e-39f7-4de5-a3a3-2066431b9ca8" |


**Response Codes**

- 200 OK - **Access Token is valid**
- 404 Not Found - **If your credentials are not valid**

{
 "code": 404,
 "message": "Invalid partner credentials."
}
- 404 Not Found - **If access token is not valid**

{
 "code": 1404,
 "message": "Invalid access token."
}
- 500 Internal Error - **If any other internal error**

{
 "code": 500,
 "message": "Fail to validate the token"
}


IMPORTANT: Truecaller SDK already verifies the authenticity of the response before forwarding it to the your app.

#### C. Request-Response correlation check

Every request sent via a Truecaller app that supports truecaller SDK 0.7 has a unique identifier. This identifier is bundled into the response for assuring a correlation between a request and a response. If you want you can check this correlation yourself by:

1. You can use your own custom request identifier via the TrueClient with `TrueSDK.getInstance().setRequestNonce(customHash);`
2. In `ITrueCallback.onSuccesProfileShared(TrueProfile)` verify that the previously generated identifier matches the one in TrueProfile.requestNonce.

IMPORTANT: Truecaller SDK already verifies the Request-Response correlation before forwarding it to the your app.


### Error Codes
