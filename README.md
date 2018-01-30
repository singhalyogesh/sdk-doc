# Android SDK

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
2. Add the provided truesdk-0.6.aar file into your libs folder. Example path: /app/libs/ ;
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

6. Add the TrueButton view in the selected layout:

    ```java
    <!--
    You can have only one TrueButton per Activity
    Android Studio should offer auto complete for the truesdk:truebutton_text values
    The possibilities are:
    truesdk:truebutton_text="autoFill"
    truesdk:truebutton_text="autoFillShort"
    truesdk:truebutton_text="signIn"
    truesdk:truebutton_text="signInShort"
    truesdk:truebutton_text="signUp"
    truesdk:truebutton_text="signUpShort"
    truesdk:truebutton_text="register"
    truesdk:truebutton_text="registerShort"
    truesdk:truebutton_text="cont"
    truesdk:truebutton_text="contShort"
    Defaults to "autoFill"
    -->
    <com.truecaller.android.sdk.TrueButton
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    truesdk:truebutton_text="autoFill"/>
    ```

7. In your selected Activity

   - Either make your Activity implement ITrueCallback or create an instance. This interface has 2 method: onSuccesProfileShared(TrueProfile) and onFailureProfileShared(TrueError).
    
   - Create an instance of TrueClient in the onCreate method:
    
     ```java
     mTrueClient = new TrueClient(Context, ITrueCallback);
     ```
    (Optional) You can set an unique requestID for every profile request with `mTrueClient.setRequestNonce(customHash);`
	- Provide to the TrueButton the created TrueClient:

      ```java
      ((TrueButton) findViewById(R.id.com_truecaller_android_sdk_truebutton)).setTrueClient(mTrueClient);
      ```
    
      The TrueButton knows whether it is usable or not: ((TrueButton) findViewById(R.id.com_truecaller_android_sdk_truebutton)).isUsable(); This can be used as a strategy for hiding the button.
   - Add in the onActivityResult method the following condition:

      ```java
      if (mTrueClient.onActivityResult(requestCode, resultCode, data)) {
          return;
      }
      ```
   - Write all the relevant logic in onSuccesProfileShared(TrueProfile) for displaying the information you have just received and onFailureProfileShared(TrueError) for handling the error and notify the user.

  	 (Optional)  
     In other to use a custom button instead of the default TrueButton call mTrueClient.getTruecallerUserProfile() in its onClick listner. Make sure your button follow our visual guidelines.

### Advanced and Optional

#### A. Server side Truecaller Profile authenticity check

Inside TrueProfile class there are 2 important fields, payload and signature. Payload is a Base64 encoding of the json object containing all profile info of the user. Signature contains the payload's signature. You can forward these fields back to your backend and verify the authenticity of the information by:

1. Fetch Truecaller public keys using this api: https://api4.truecaller.com/v1/key (you need to fetch the keys only if you have never done it or if you cannot verify the signature with any of the already cached keys);
2. Loop through the public keys and try to verify the signature and payload;

IMPORTANT: TrueSDK already verifies the authenticity of the response before forwarding it to the your app.

#### B. Request-Response correlation check

Every request sent via a Truecaller app that supports TrueSDK 0.6 has a unique identifier. This identifier is bundled into the response for assuring a correlation between a request and a response. If you want you can check this correlation yourself by:

1. Generate the request identifier via the TrueClient: `mTrueClient.generateRequestNonce();` or use your own Nonce set with `mTrueClient.setRequestNonce(customHash);`
2. `In ITrueCallback.onSuccesProfileShared(TrueProfile)` verify that the previously generated identifier matches the one in TrueProfile.requestNonce.

IMPORTANT: TrueSDK already verifies the Request-Response correlation before forwarding it to the your app.
