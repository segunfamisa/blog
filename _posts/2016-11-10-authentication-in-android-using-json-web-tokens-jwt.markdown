---
layout: post
title: "Authentication in Android using JSON Web Tokens (JWT)"
description: "Learn how to implement authentication with JWT in your native Android app"
date: 2016-11-10 10:34
author:
  name: "Segun Famisa"
  url: "https://twitter.com/segunfamisa"
  mail: "segunfamisa@gmail.com"
  avatar: "https://2.gravatar.com/avatar/9ab0b3b080e75e0c03a0c643333f8b93"
design:
  bg_color: "rgb(150, 194, 49)"
  image: https://i.imgur.com/4ED5Esq.png
tags:
- android
- jwt
- authentication
---

**TL;DR summary:** Authentication is a big deal for your app. JWT allows you safely transfer information between your app and server and helps you identify who’s making the requests. This post shows how to implement JWT in your native Android app. If you’re in a hurry and just want to dive in code, clone the final code from the [Github repository](https://github.com/segunfamisa/android-jwt-authentication).

---

## Introduction

Authentication is an important part of applications that involves the need to establish identity of an entity requesting information before honouring such requests. There are quite a number of techniques, protocols, standards of authenticating users in your app.

A [JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519) is a standard way of representing information (referred to as claims) to be transferred between two parties. JWTs are commonly used for authentication purposes because it enables your server identify that a user is who they say they are before serving their requests for resources. It also provides a way of  securely transferring information.

If you already know the parts that make up a JWT, you can dive straight into how to implement JWT in your native Android app.

A typical JWT has three parts, delimited by dots (`.`), in the format: **_`header.payload.signature`_**


The **_header_** part of a JWT contains information related to the type of token and the algorithm used to generate it. Examples of the algorithms include [HMAC SHA 256](https://tools.ietf.org/html/rfc4868), [HMAC SHA 512](https://tools.ietf.org/html/rfc4868) among others. This part is formed by doing a [Base64](https://tools.ietf.org/html/rfc4648) encoding of the JSON string containing the information above.


The **_payload_** is like the “body” of the JWT and contains the claims to be transferred. Claims are information about the entity and which may be used to identify the entity making the request. For example, your claim could be the name of the user using the app or/and the role of such user.
The payload part is formed by doing a Base64 encoding of the JSON string that contains such information.


The **_signature_** is the 3rd part and it is built by concatenating the encoded forms of the header and the payload by a dot (.) and signing with the algorithm specified in the header, using a secret.


Stringing all three parts together like `header.payload.signature` gives us a JSON Web Token, which we can now go ahead and use in applications.

This is just a quick intro into the structure of a JWT, be sure to check out https://jwt.io/ for more information on JWTs.

## Implementing JWTs on Android

Now that we know what exactly a JWT is, and how we can build one, let’s go ahead and build an Android app that requires JWT for authentication.

We’re going to use the [NodeJS JWT Authentication API](https://github.com/auth0-blog/nodejs-jwt-authentication-sample) to build an app that can allow a user sign up or login and then get exclusive protected quotes from the great Chuck Norris.

Fun quote:
>_When Chuck Norris does a pushup, he isn’t lifting himself up, he’s pushing the Earth down :D_

### 0. Setup
Anyway, let’s get started, but before you do that, be sure you have the following requirements:

  * [Android Studio (1.5 or above)](https://developer.android.com/studio/index.html)
  * Emulator running API 15 or above.

We also need to set up the API server and to do that, go ahead and clone the server repository [here](https://github.com/auth0-blog/nodejs-jwt-authentication-sample) and start it.
You can make use of the following commands (requires that you have Node JS installed on your machine):

```bash
git clone https://github.com/auth0-blog/nodejs-jwt-authentication-sample.git
cd nodejs-jwt-authentication-sample/
npm install
node server.js
```

**Note:** the server runs by default at: localhost:3001 by default.

### 1. Create Project
Create a new project using Android studio. Go to `File -> New -> New project` and follow the setup wizard.

### 2. Add Internet permission
Since our app will to need to access the internet, we need to add the internet permission to our AndroidManifest.xml file. We will do that by adding the following line:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### 3. Add dependencies
We’ll be using a couple of dependencies for this app. Some of which includes [OkHttp](http://square.github.io/okhttp/) - one of the most popular Http clients in Android and [Gson](https://github.com/google/gson), a library to deserialize JSON strings into Java objects and serialize Java objects into JSON strings.

We will also be using auth0’s [JWT library for Java](https://github.com/auth0/java-jwt), to avoid writing boilerplate code when we need to verify our JWT.

To add these dependencies, navigate to your app-module’s build.gradle file (`<project-name>/<app-module>/build.gradle`) and add the following lines to your `dependencies` block:

```gradle
...
dependencies {
    ...
    compile 'com.auth0:java-jwt:2.2.1'
    compile 'com.squareup.okhttp3:okhttp:3.4.1'
    compile 'com.google.code.gson:gson:2.4'
}
```

### 4. Build the Login/Signup UI
Create a new empty activity and name it `LoginActivity` and the layout file `activity_login.xml` by following the screenshots below:

![Create new activity](https://i.imgur.com/YbK1bMR.png)

![Create new activity details](https://i.imgur.com/RVTXony.png)

The login UI for our app is simple, we have a single layout, but we hide/show fields depending whether the action is to login, as we have in the screenshot below.

![](https://i.imgur.com/qbRjRWn.png)

### 5. Login & Signup
Now that we’ve created the UIs, next, we want to make API requests to log the user in, or create a new user. To do this, we send the user’s credentials to the server in a `POST` request and the server sends us a JWT back. We will then use this JWT to access the protected quotes from the great one _Chuck Norris._

  * Since we’re using OkHttp to make requests and Gson to deserialize the JSON response into JSON objects, we need to create the Java class to match the JSON response for login and sign up.

  We can do this using [http://www.jsonschema2pojo.org/](http://www.jsonschema2pojo.org/) or we can manually build that class. Either way we choose, we end up with a class looking like:

```java
public class Token {
   @SerializedName("id_token")
   private String idToken;

   public String getIdToken() {
       return idToken;
   }
}
```
  * We also need to create a Callback interface that we’ll implement in order to retrieve the response from a successful network call or an error.

```java
/**
 * Callback interface for network response and error
 * @param <T> class representing the response
 */
 public interface Callback<T> {
    void onResponse(@NonNull T response);
    void onError(String error);
    Class<T> type();
 }
```

  * To make the request to log the user in, let’s create a method that accepts a username, password and a callback we will implement to retrieve the JWT when the call is successful. The following block of code explains how this is done:

```java
  private void doLogin(String username, String password, final Callback callback) {
     String url = "http://10.0.2.2:3001/sessions/create";
     HttpUrl httpUrl = HttpUrl.parse(url);

     // create the request body
     FormBody.Builder body = new FormBody.Builder();
     body.addEncoded("username", username);
     body.addEncoded("password", password);

     // create the request
     Request request = new Request.Builder()
             .url(httpUrl)
             .post(body.build())
             .build();

     // create client and make a call
     OkHttpClient client = new OkHttpClient();
     client.newCall(request)
             .enqueue(new okhttp3.Callback() {
                 // get a handler for the UI thread
                 Handler mainHandler = new Handler(Looper.getMainLooper());
                 @Override
                 public void onFailure(Call call, final IOException e) {
                     if (callback != null) {
                         mainHandler.post(new Runnable() {
                             @Override
                             public void run() {
                                 // invoke error callback in case of error
                                 callback.onError(e.toString());
                             }
                         });
                     }
                 }

                 @Override
                 public void onResponse(final Call call, final Response response) {
                     if(callback != null) {
                         try {
                             final String stringResponse = response.body().string();

                             // use gson to serialize the string respones to a java object
                             final Token token = new Gson().fromJson(stringResponse, Token.class);
                             mainHandler.post(new Runnable() {
                                 @Override
                                 public void run() {
                                     // invoke response callback in case of successful response
                                     callback.onResponse(token);
                                 }
                             });
                         } catch (final IOException ioe) {
                             mainHandler.post(new Runnable() {
                                 @Override
                                 public void run() {
                                     callback.onError(ioe.toString());
                                 }
                             });
                         }
                     }
                 }
             });
  }
```

**Note:**  

  * We use the `Looper.getMainHandler()` to be able to post updates to the main thread, since we are likely going to update the UI from a background thread.
  * 10.0.2.2 is used here instead of 'localhost' because that's the IP Address through which an emulator can access the localhost of the host machine. If you are testing on a real Android device, you can use the local IP Address of the development machine (like 192.168.0.2), but your phone must be connected to the same network.

To use this `doLogin()` method, we will call it in the `onClick` of the login button:

```java
...
mButtonAction.setOnClickListener(new View.OnClickListener() {
   @Override
   public void onClick(View view) {
       String username = mEditUsername.getText().toString().trim();
       String password = mEditPassword.getText().toString().trim();

       doLogin(username, password, loginCallback);
   }
});
```

We then implement the Callback like:

```java
...
Callback loginCallback = new Callback<Token>() {
    @Override
    public void onResponse(@NonNull Token response) {
        // save token to shared preferences and navigate to the quotes activity
        saveToken(response);
        startActivity(new Intent(LoginActivity.this, QuotesActivity.class));
    }

    @Override
    public void onError(String error) {
        Toast.makeText(LoginActivity.this, error, Toast.LENGTH_SHORT).show();
    }

    @Override
    public Class<Token> type() {
        return Token.class;
    }
}
```

**Note:** for sign up, we do something similar, the only difference being the addition of an extra parameter “extra” according to the API.

### 6. Build the Quotes UI
The UI for the Quotes is pretty simple. We just want a `TextView` to show the welcome message, to personalize the experience, a `TextView` to display the quote, and a `Button` to fetch a new quote.
To do this, create an activity similar to how we did in step 4 above. The screenshot below shows the UI we're trying to achieve.

The XML code to create this UI is as we have in the [repository](https://github.com/segunfamisa/android-jwt-authentication/blob/master/app/src/main/res/layout/activity_quotes.xml).

### 7. Get username claim from JWT
Now we have our UI setup, and we have the JWT for the session. Next thing we want to do is retrieve the `username` claim from the JWT and use it in a welcome message.
To do this, we will make use of the JWT Java library that we added earlier.

We can achieve this using the `JWTVerifier` class. We create an object of the class, passing the encryption secret as a parameter, and then calling the `verify()` method.

```java
private String getUsernameFromJWT(String token) {
    private String secret = ""; // secret is what was used to create the JWT on the server side.
    JWTVerifier verifier = new JWTVerifier(secret);
    try {
        Map<String, Object> claims = verifier.verify(token);
        if (claims != null && claims.containsKey("username")) {
            return claims.get("username").toString();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

We can now set the username that was retrieved on the welcome `TextView` like:

```java
mWelcomeText.setText(String.format("Welcome, %s", getUsernameFromJWT()));
```

### 8. Get protected quotes
Next thing we want to do is request for protected quotes using the JWT we received and saved in step 5 above.

To fetch a protected quote, we make a GET request to `/api/protected/random-quote` while supplying the JWT in the `Authorization` field of the header in the format: `Authorization: Bearer <JWT>`.

To implement this, we will need a method like:

```java
private void doGetQuote(String token, Callback callback) {
    String url = "http://10.0.2.2:3001/api/protected/random-quote";
    HttpUrl httpUrl = HttpUrl.parse(url);

    // build request object and add token as header
    Request request = new Request.Builder()
            .url(httpUrl)
            .addHeader("Authorization", "Bearer " + token)
            .get()
            .build();

    // create client and make a call
    OkHttpClient client = new OkHttpClient();
    client.newCall(request)
            .enqueue(new okhttp3.Callback() {
                // get a handler for the UI thread
                Handler mainHandler = new Handler(Looper.getMainLooper());
                @Override
                public void onFailure(Call call, final IOException e) {
                    if (callback != null) {
                        mainHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                // invoke error callback in case of error
                                callback.onError(e.toString());
                            }
                        });
                    }
                }

                @Override
                public void onResponse(final Call call, final Response response) {
                    if(callback != null) {
                        try {
                            final String stringResponse = response.body().string();
                            mainHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    // invoke response callback in case of successful response
                                    // the request for a quote returns a string response, so no need to convert to Java object
                                    callback.onResponse(stringResponse);
                                }
                            });
                        } catch (final IOException ioe) {
                            mainHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    callback.onError(ioe.toString());
                                }
                            });
                        }
                    }
                }
            });
}
```

This method is then called in the `onClick` method of the get quote button:

```java
    mButttonRandomQuote.setOnClickListener(new View.OnClickListener {
        @Override
        public void onClick(View view) {
            String token = getTokenFromPreferences();
            doGetQuote(token, getQuoteCallback);
        }
    });
```

We then implement the getQuoteCallback as follows:

```java
    Callback getQuoteCallback = new Callback<String>() {
        @Override
        public void onResponse(@NonNull String quote) {
            mTextQuote.setText(quote);
        }

        @Override
        public void onError(String error) {
            Toast.makeText(QuotesActivity.this, error, Toast.LENGTH_SHORT).show();
        }

        @Override
        public Class<String> type() {
            return String.class;
        }
    };
```

## Bonus: save some time with auth0.
Phew! if you followed through the steps 0-8, you'll see that securing your apps using JWTs on Android isn't exactly a piece of cake.
In fact, things can get really complicated and quickly too.

Good news is that you can achieve all of this, in fewer easy steps using auth0's authentication libraries. Let's take a look at how that can happen with [auth0's Android Lock library](https://github.com/auth0/Lock.Android):

  * Get an account on auth0 at [https://auth0.com/](https://auth0.com/).
  * Create a new client following the screenshot below:
  ![Create new client](https://i.imgur.com/RGwBx11.png)

  * Configure callback URLs on the auth0 dashboard

  Select the `Settings` tab in your client dashboard, and configure the callback URL for your app. The callback URL is generated using your auth0 app domain and your app's package name in the format:

  `https://<domain>.auth0.com/android/<app-package-name>/callback`

  See the image below for more information:

  ![Configure callback urls](http://i.imgur.com/PMDd42d.png)

  * Add auth0 dependencies

  Next step, after configuring the app on the dashboard, is to add the auth0 dependencies to your app-module `build.gradle` file:

  ```gradle
    ...
    dependencies {
        ...
        compile 'com.auth0.android:lock:2.1.1'
    }
  ```
  * Setup credentials

  To get it working, we need to setup the credentials in our app. To do this, we set the auth0 domain as well as auth0 client id and also setup the `AndroidManifest.xml`.

  i. Add the following lines to your `strings.xml` file

  ```xml
  <resources>
    ...
    <string name="auth0_client_id">YOUR AUTH0 CLIENT ID</string>
    <string name="auth0_domain">YOUR AUTH0 DOMAIN</string>
  </resources>    
  ```

  ii. Configure the `AndroidManifest.xml` file

  We need to add the auth0 lock Activity to the manifest file, and setup the credentials we created earlier. To do this, we add the following lines to the <application> tag of our `AndroidManifest.xml` file:

  ```xml
  <application
      ...
      <uses-permission android:name="android.permission.INTERNET" />
      <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
      >
      ...
        <!-- begin auth0 lock activity -->
        <activity
            android:name="com.auth0.android.lock.LockActivity"
            android:label="@string/app_name"
            android:launchMode="singleTask"
            android:screenOrientation="portrait"
            android:theme="@style/Lock.Theme">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <data
                    android:host="@string/auth0_domain"
                    android:pathPrefix="/android/<PACKAGE NAME>/callback"
                    android:scheme="https" />
            </intent-filter>
        </activity>
        <!-- end lock activity -->

        <!-- begin auth0 webauth activity -->
        <activity
            android:name="com.auth0.android.provider.WebAuthActivity"
            android:theme="@style/Lock.Theme" />
        <!-- end auth0 webauth activity -->
  </application>
  ```

  * Retrieve token

  Now that we've configured all we need to configure, next step is to log the user in and retrieve a token for the user.
 To do this, we make use of the lock library.

 In the `onCreate` method of your activity, you initialize the auth0 lock library by doing something like:

 ```java
Auth0 auth0 = new Auth0("YOUR_AUTH0_CLIENT_ID", "YOUR_AUTH0_DOMAIN");
lock = Lock.newBuilder(auth0, callback)
   //Customize Lock
   .build(this);

//start the lock activity
startActivity(mLock.newIntent(this));
 ```

 We initialize and assign the `callback` as:

 ```java
private final LockCallback mCallback = new AuthenticationCallback() {
     @Override
     public void onAuthentication(Credentials credentials) {
         // save credentials and navigate to protected activity
     }

     @Override
     public void onCanceled() {
         Toast.makeText(getApplicationContext(), "Log In - Cancelled", Toast.LENGTH_SHORT).show();
     }

     @Override
     public void onError(LockException error) {
         Toast.makeText(getApplicationContext(), "Log In - Error Occurred", Toast.LENGTH_SHORT).show();
     }
 };
 ```

   * For more info about how auth0 can help save time when handling authentication, check out the [quick start](https://auth0.com/docs/quickstart/native/android/00-introduction) and [sample codes](https://github.com/auth0-samples/auth0-android-sample)


## Conclusion

As we have seen, authentication is a really important part of our apps.
We have seen and learnt what JWTs are made up of and how to implement them in Android. JWT helps to protect resources by ensuring that the user requesting for these resources are who they say they are. While doing this, JWTs also allow us transfer information between the server and the app using the `claim` fields in the JWT.
