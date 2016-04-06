---
layout: post
title: Making Android application with advertising and in-app billing
category: Android
tags:
---
![billing and ad test](http://lh6.ggpht.com/K44x_0coTYPeL_1InJFk0876iQisXAB6Hfx9JvRKxNHVGfC-VhQ128WZyopx40gfiZhb-v1R-PxmE-hYsxSsBhLpMQ=s700) 

Hi folks, today I gonna show you how to make Android application which will display an advertising and will use Google Play in-app billing to buy a premium status to disable that "annoying" advertising. Please go to my [github repository](https://github.com/denisigo/android-billingadtest) to get the source code.

<!--more-->

# How it works

Both advertising and in-app billing use Google Play Services application which is basically pre-installed on the Android devices. To build applications which use Google Play Services we need to install Google Play Services SDK.

## Advertising

To display advertising we will use [Google Mobile Ads SDK](https://developers.google.com/mobile-ads-sdk/), which is now offered through [Google Play Services](https://developer.android.com/google/play-services/ads.html). Please note that Google has dropped Google Play Services support for Android 2.2 Froyo and older, so, you will not be able to display ads on that devices. 

Also, you have to create [AdMob](http://admob.com) Publisher account and special ad unit ID (on the AdMob publisher console) which we will use in our app to identify ad banner.

## In-app billing

To use in-app billing we will use Google Play Services' [billing API](https://developer.android.com/google/play/billing/index.html). Google also offers [sample app](https://developer.android.com/training/in-app-billing/preparing-iab-app.html), which provides handy helper library and we will use it to deal with billing API. 

To sell in-app products (premium status is the virtual in-app product) we have to create Google Wallet Merchant account. 

**Please note, that to be able to test in-app billing in your app you must have another Google account (different from your publisher one). You must (at least at the time writing) set this account as primary or sole Google account on your testing device or AVD.**

# Registering accounts

At first, let's make sure you have registered Google Publisher and Google Wallet Merchant accounts - see instructions [here](http://developer.android.com/distribute/googleplay/publish/register.html). 

Next, let's register [AdMob Publisher account](https://apps.admob.com/admob/signup). It uses your existing Google account, so select one. 

Then go to the AdMob [console](https://apps.admob.com/#home) and click on “+ Monetise new app” button: 

[![android billing and ad app](http://lh6.ggpht.com/j_1bNJkqEGWsnZmFVCd08J61zj1yHMgJQvbaLr4PXgeJAaxr8SLlbKC5h9WTQUxv2QV1GLHKFKDQmAT5-UrKbLOtoQ=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app.jpg) 

Then select “Add your app manually”, enter app name and select platform: 

[![android billing and ad app 2](http://lh3.ggpht.com/LqFDQSdaemjyPBUkAdSlsDQMfz0VT7ipcqZ2Qff4EMSZ5HhL9ENBva5R5Li6bUqlBSA01q6B2nNG2_OjzJsgLwEEdQ=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app-2.jpg) 

Then select “Banner” type, and enter ad unit name. Leave other options as is if you’re not sure. We can change it later if we want. Click “Save” button: 

[![android billing and ad app 3](http://lh6.ggpht.com/zo6bqPS-NKOdTeY6DrZY2i8cSoSMWDgrVYp0T-B3ith2tvLcWq5VgloDFUbTGxNvBFM_kYBne0UF6-RAq-ITHl8c=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app-3.jpg)

Locate and store somewhere “Ad unit ID” string, we will need it later: 

[![android billing and ad app 4](http://lh6.ggpht.com/cZlJY8-ltCNK4TnB2oFwo9aq6fQvEBUildZrXBPjX6MKPjOz4mnOrZdXrnvk_zi_xjm6eObulr_MvoeJnSuitOY=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app-4.jpg) 

Click “Done”. That’s all for a while. Remember URL of the console - you will find there statistics about advertising when the app is published.

# Setting testing account for in-app billing

As I said earlier, Google wants us to have another account for testing in-app billing. Assuming you've already created it, let's add it to the right place. In the [Google Play Developer console](https://play.google.com/apps/) go to the **Settings > Account details > License testing** and add your test Google account's email: 

[![android billing and ad app 5](http://lh5.ggpht.com/IQoyJZ0SFP-NBZTv-8ZshTunLf0m2LYfbFBNWsD9QBlDnElIe_tMQMxQKYCgJuvJ6i6Lj3XafEAm6h9y7bCIkxDzVA=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app-5.jpg) 

Click Save and we're done.

# Installing Google Play Services SDK and Google Play Billing library

Installing Google Play Services SDK is pretty good described [here](https://developer.android.com/google/play-services/setup.html), and installings Google Play Billing library is described [here](https://developer.android.com/training/in-app-billing/preparing-iab-app.html). But let's describe it in a glance with screenshots =). 

At first, open (I use Eclipse) Android SDK manager (Window > Android SDK Manager), scroll down to the Extras list, expand it and select Google Play Services and Google Play Billing library and click Install packages button: 

[![installing google play services](http://lh3.ggpht.com/HlGzwlYE9wVyhQYrohH0OkmFp_jbX2TCKqA53SaNNKontF7a1knrIieSiC5WiPZCNTFJdkq4L7EsSF7jzTolBRbz=s700)](http://denisigosite.appspot.com.storage.googleapis.com/installing-google-play-services.jpg) 

Now, files of Google Play Services library reside here: **[android-sdk]/extras/google/google_play_services/**, files of Google Play Billing library reside here: **[android-sdk]/extras/google/play_billing/**. 

Billing library doesn't need any work for a while, so let's continue with Services library. 

At first, copy Google Play Services library project directory **[android-sdk]/extras/google/google_play_services/libproject/google-play-services_lib/** into your workspace directory. Then import it into your Eclipse workspace: open File > Import > Android > Existing Android code into workspace and browse to the directory you've copied.

# Creating an app project

Now, you can create a new application or use existing one. Let's assume we've created a new blank Android application (File > New > Android Application Project). Don't forget to set right package name! 

So, let's continue.

# Setting up Google Play Services library

For our app to be able to work with Google Play Services we need to [reference](https://developer.android.com/tools/projects/projects-eclipse.html#ReferencingLibraryProject) Google Play Services library in our app. So, just open your project's Properties, select Android tab, click Add button and select "google_play_services_lib":

[![installing google play services 2](http://lh5.ggpht.com/XinxuxBe681f-5TV1nebUa_2390dPtg7HSiKoB3TkQI1TIrncvCnVSfOnu30KPxUzbpciu3sSCfZxgvtq2Cy_QFwkw=s700)](http://denisigosite.appspot.com.storage.googleapis.com/installing-google-play-services-2.jpg) 

Next, let's add the following tag to the app's manifest file as a child of <application> tag:

``` xml
<meta-data android:name="com.google.android.gms.version"
           android:value="@integer/google_play_services_version" />
```

Add the following lines to the proguard-project.txt:

```
-keep class * extends java.util.ListResourceBundle {
    protected Object[][] getContents();
}

-keep public class com.google.android.gms.common.internal.safeparcel.SafeParcelable {
    public static final *** NULL;
}

-keepnames @com.google.android.gms.common.annotation.KeepName class *
-keepclassmembernames class * {
    @com.google.android.gms.common.annotation.KeepName *;
}

-keepnames class * implements android.os.Parcelable {
    public static final ** CREATOR;
}
```

And don't forget to add/uncomment the following line in the project.properties file:

```
proguard.config=${sdk.dir}/tools/proguard/proguard-android.txt:proguard-project.txt
```

# Setting up advertising

To show advertising we have to update our app's manifest file. At first, we have to request permission to access the internet:

``` xml
<uses-permission android:name="android.permission.INTERNET" />
```

Then, we have to add activity required to show ad overlays:

``` xml
<activity
            android:name="com.google.android.gms.ads.AdActivity"
            android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|uiMode|screenSize|smallestScreenSize"/>
```

Now we have to prepare placement for our banner - just add RelativeLayout on the top of your main activity layout:

``` xml
<RelativeLayout
        android:id="@+id/adplacement"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:visibility="gone" />
```

Add "ad unit ID" string we've got from AdMob to the values/strings.xml:

```xml
<string name="ad_unit_id">ca-app-pub-123123123123/123123123</string>
```

# Setting up in-app billing

As I said earlier, Google provides sample "Trivial driving" app which contains helper library which we will use. Also, using in-app billing requires special package containing in-app billing service interface definition. So, let's do it. To add them it would be much easier to import sample app into your workspace firstly - simply import existing project as we described above from following location: **[android-sdk]/extras/google/play_billing/samples/TrivialDrive**. Then, open **TrivialDrivingBillingTest** app from your package explorer and copy both **com.android.vending.billing** and **com.example.android.trivialdrivesample.util** packages to your app's src directory right in the package explorer. Then, rename your copy of **com.example.android.trivialdrivesample.util** using your app package name. In my case it is **com.denisigo.billingadtest.billingutil**. 

Now, we have to request billing permission in the manifest file:

``` xml
<uses-permission android:name="com.android.vending.BILLING" />
```

# Preparing first release

Now, it's almost time to go to Google Play Developer console and add in-app product for premium status. But, Google Play Developer console wants us to upload our app APK first. So, let's compile our app (if everything is going right, it should compile without errors) in the release mode and sign it with our **developer** certificate. Eclipse has handy wizard to perform this step - just select your project in the package explorer and open File > Export... > Export Android Application. **Be careful, because if you upload your app signed with debug certificate, you won't be able to upload one signed with developer certificate next time.**

# Adding our app in the Google Play Developer console

Let's create a new application. Go to the Developers's console home screen and click **Add new application** button. Set some name and click **Upload APK** button. Don't forget to upload your release APK!

# Setting up in-app products

Let's assume you've successfully created a new application, so it's time to add in-app product for our premium status. Open your app from the list, go to the **In-app Products** tab and click **Add new product** button. Select **Managed product**, set **premium** as **Product ID** and click **Continue**. [![android billing and ad app 6](http://lh4.ggpht.com/amDMhSG7-qHAOlosXjJ3Db1i-BUFT09ztLgu5nYGM2wQ1tCEi6PZL_W8Iu41rBJgvqOeqkqTrWz0pA1uH8QMe2s=s700)](http://denisigosite.appspot.com.storage.googleapis.com/android-billing-and-ad-app-6.jpg) In the window opened you have to enter some name, description and set price for this product. Remember, that for test accounts we've set before, money will be refunded. We have to do one more thing here - go to the **Services & APIs** tab, scroll down and copy base64-encoded string named **YOUR LICENCE KEY FOR THIS APPLICATION**.

# Let's code!

I won't place here all the app code, but instead I'll explain here main concepts. Full code of this app you can find in my [github repository](https://github.com/denisigo/android-billingadtest).

## In-app billing

**Warning: To simplify the code I've omitted some important security things, which might be crucial to your app, so please read more about it [here](https://developer.android.com/google/play/billing/billing_best_practices.html).** At first, let's declare our premium product ID, we set before in the Play Developer console:

``` java
private static final String SKU_PREMIUM = "premium";
```

Now, let's create IabHelper instance in the onCreate method and invoke its onSetup method:

``` java
// Create in-app billing helper
mAbHelper = new IabHelper(this, base64EncodedPublicKey);
// and start setup. If setup is successfull, query inventory we already own
mAbHelper.startSetup(new IabHelper.OnIabSetupFinishedListener() {
	public void onIabSetupFinished(IabResult result) {

		if (!result.isSuccess()) {
			return;
		}

		mAbHelper.queryInventoryAsync(mGotInventoryListener);
	}
});
```

Please notice **mAbHelper.queryInventoryAsync** call. It will be called in case setup passed successfully and we can query inventory for the current user (Google account active on the device). This is necessary because user might already has premium status and we should check it at the startup to disable advertising. If user doesn't have premium status and presses **Upgrade To Premium**, so called purchase flow will be started:

``` java
public void onBtUpgradeClick(View view) {
	mAbHelper.launchPurchaseFlow(this, SKU_PREMIUM, 0,
		mPurchaseFinishedListener, "");
}
```

Please notice **mGotInventoryListener** and **mPurchaseFinishedListener** listeners. They are invoked when either **queryInventoryAsync** or **launchPurchaseFlow** methods complete successfully. In both cases id check/purchase was successful, we set **mIsPremium** flag and update the interface to show or hide the advertising:

``` java
mIsPremium = true;
updateInterface();
```

## Advertising

As you might already noticed, in the **onCreate** method we've got RelativeLayout instance for our ad banner:

``` java
mAdLayout = (RelativeLayout) findViewById(R.id.adplacement);
```

Now, let's look at **displayAd** method which is being invoked from **updateInterface** method:

``` java
private void displayAd(boolean state) {

	if (state) {
		if (mAdView == null) {

			// Google has dropped Google Play Services support for Froyo
			if (Build.VERSION.SDK_INT > Build.VERSION_CODES.FROYO) {
				mAdLayout.setVisibility(View.VISIBLE);

				mAdView = new AdView(this);
				mAdView.setAdUnitId(getResources().getString(
						R.string.ad_unit_id));
				mAdView.setAdSize(AdSize.BANNER);

				RelativeLayout.LayoutParams params = new RelativeLayout.LayoutParams(
							RelativeLayout.LayoutParams.MATCH_PARENT,
							RelativeLayout.LayoutParams.WRAP_CONTENT);
				mAdLayout.addView(mAdView, params);
				mAdView.loadAd(new AdRequest.Builder().build());
			}
		}
	} else {

		mAdLayout.setVisibility(View.GONE);
		if (mAdView != null) {
			mAdView.destroy();
			mAdView = null;
		}
	}
}
```

As you can see, we must add Android version check to prevent our app from crashes on older devices. Then we just create **AdView** instance by passing there our **Ad unit ID** we've placed to the values/strings.xml before. And then we just add new view to our RelativeLayout we've got in **onCreate**. That's all I hope.

# Testing

To test our app we should again compile it in the release mode and upload it to the Google Play Developer console. In case you previously launched your app in debug mode, you should uninstall it at first. You can uninstall it by adb by specifying your package name:

``` shell
adb uninstall com.denisigo.billingadtest
```

Then you should install release APK to your device with adb by specifying your APK file name:

``` shell
adb install -r billingadtest.apk
```

In case you have multiple devices connected, you should update these commands with -s flag to specify particular device like this:

``` shell
adb -s SH09PPL02591 install -r billingadtest.apk
```

Where SH09PPL02591 is device's ID which you can find by following adb command:

``` shell
adb devices -l
```

OMG, that's all! o_O
