# Xsolla Mobile SDK for Android

![License](https://img.shields.io/github/license/xsolla/xsolla-sdk-android)
![Latest release](https://img.shields.io/github/v/release/xsolla/xsolla-sdk-android)
![Android API 24+](https://img.shields.io/badge/API-24%2B-blue.svg)

## Overview

Xsolla Mobile SDK for Android provides a Google Play Billing-compatible API for integrating
in-game payments into your app via Xsolla Pay Station. It mirrors Google's Billing Library
patterns (`BillingClient`, `ProductDetails`, `Purchase`) so integration feels familiar to
Android developers.

Key features include 1000+ payment methods across 200+ geographies, 130+ currencies,
built-in anti-fraud protection, 25+ languages, player authentication (Xsolla Login widget,
social login, custom tokens), product catalog and virtual items, and Buy Button / Web Shop
integration.

## Requirements

- Android API 24+
- Java 11+
- A Xsolla Publisher Account ([publisher.xsolla.com](https://publisher.xsolla.com))

## Install

Add the Xsolla Maven repository to your `settings.gradle`:

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()

        maven {
            url "https://raw.githubusercontent.com/xsolla/xsolla-sdk-android/main"
        }
    }
}
```

Then add the dependency in your module's `build.gradle`:

```groovy
dependencies {
    implementation 'com.xsolla.android:mobile:3.0.45'
}
```

## Usage

Configure the SDK and start a billing connection:

```java
import com.xsolla.android.mobile.*;

int PROJECT_ID = 77640;
String LOGIN_ID = "026201e3-7e40-11ea-a85b-42010aa80004";

Config config = new Config(
    Config.Common.getDefault()
        .withSandboxEnabled(true),
    Config.Integration.forXsolla(
        Config.Integration.Xsolla.Authentication.forAutoJWT(
            ProjectId.parse(PROJECT_ID).getRightOrThrow(),
            LoginUuid.parse(LOGIN_ID).getRightOrThrow()
        )
    ),
    Config.Payments.getDefault(),
    Config.Analytics.getDefault()
);

BillingClient billingClient = BillingClient.newBuilder(context)
    .setListener(purchasesUpdatedListener)
    .setConfig(config)
    .build();

billingClient.startConnection(new BillingClientStateListener() {
    @Override
    public void onBillingSetupFinished(BillingResult billingResult) {
        if (billingResult.getResponseCode() == BillingClient.BillingResponseCode.OK) {
            // Ready to query products and make purchases
        }
    }

    @Override
    public void onBillingServiceDisconnected() {
        // Handle reconnection
    }
});
```

## Documentation

Full integration guide, API reference, and SDK Explorer:
[developers.xsolla.com/sdk/](https://developers.xsolla.com/sdk/)

Interactive demo: [developers.xsolla.com/sdk/demo/](https://developers.xsolla.com/sdk/demo/)

## Support

- [GitHub Issues](https://github.com/xsolla/xsolla-sdk-android/issues)
- [Xsolla Developer Portal](https://developers.xsolla.com/)

## License

MIT License. See [LICENSE](./LICENSE).
