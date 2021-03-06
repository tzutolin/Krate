# Krate

[![Build Status](https://app.bitrise.io/app/40d6bd22db4cfda8/status.svg?token=0neqv73n3TXp9F0nNxj_rA&branch=master)](https://app.bitrise.io/app/40d6bd22db4cfda8)

![Krate banner](./docs/krate.png)

_Krate_ is a `SharedPreferences` wrapper library that uses delegated properties for convenient access to `SharedPreferences` values.

Here's what its basic usage looks like, extending the provided `SimpleKrate` class:

```kotlin
class UserSettings(context: Context) : SimpleKrate(context) {

    var notificationsEnabled by booleanPref("notifications_enabled", false)
    var loginCount by intPref("login_count", 0)
    var nickname by stringPref("nickname", "")

}

val settings = UserSettings(context)
settings.loginCount = 10
Log.d("LOGIN_COUNT", "Count: ${settings.loginCount}")
```

# Dependency

You can include _Krate_ in your project from the `mavenCentral()` repository, like so:

```groovy
implementation 'hu.autsoft:krate:0.4.0'
```

# Optionals vs defaults

Each stored property can be declared with or without a default value. Here's how the two options differ:

### Optional values:

A property declared with the one-argument delegate function will have a nullable type. It will have a `null` value if no value has been set for this property yet, and its current value can be erased from `SharedPreferences` completely by setting it to `null`.

```kotlin
var username: String? by stringPref("username")
```

### Default values:

A property declared with the two-argument delegate function takes its default value as the second argument, and it will have a non-nullable type. Reading from this property will return either the value it was last set to or the default value. Setting this property will update the value stored in `SharedPreferences`. Note that there's no way to remove these values from `SharedPreferences` (although you could set it explicitly to the default value).

```kotlin
var username: String by stringPref("username", defaultValue = "admin")
```

# Custom Krate implementations

You can usually get away with extending `SimpleKrate`, as it does allow you to pass in a custom name for the `SharedPreferences` to be used to store your values in its constructor as an optional parameter. (If you pass in no `name` parameter to its constructor, it will default to using the instance returned by `PreferenceManager.getDefaultSharedPreferences(context)`.)  

However, you can also implement the `Krate` interface directly if you want to manage the `SharedPreferences` instance yourself for whatever reason - all this interface requires is a property that holds a `SharedPreferences` instance. With that, you can use the delegate functions the same way as shown above: 

```kotlin
class ExampleCustomKrate(context: Context) : Krate {

    override val sharedPreferences: SharedPreferences

    init {
        sharedPreferences = context.applicationContext.getSharedPreferences("custom_krate_prefs", Context.MODE_PRIVATE)
    }

    var exampleBoolean by booleanPref("exampleBoolean", false)
    
}
```

For simple applications, your `Activity` or `Fragment` can easily serve as a `Krate` implementation:

```kotlin
class MainActivity : AppCompatActivity(), Krate {

    override val sharedPreferences: SharedPreferences by lazy {
        getPreferences(Context.MODE_PRIVATE) // Could also fetch a named or default SharedPrefs
    }
    
    var username by stringPref("username", "")

}
```

# Validation

You can add validation rules to your Krate properties by providing an additional lambda parameter, `isValid`:

```kotlin
var percentage: Int by intPref(
        key = "percentage",
        defaultValue = 0,
        isValid = { it in 0..100 }
)
```

If this validation fails, an `IllegalArgumentException` will be thrown.

# Addons

Krate, by default, supports the types that `SharedPreferences` supports. These are `Boolean`, `Float`, `Int`, `Long`, `String` and `Set<String>`. You may of course want to store additional types in Krate. For some common types, addon libraries are available, as described below.

If you don't find support for the type you're looking for, implementing your own delegate in your own project based on the code of existing delegates should be quite simple, this is very much a supported use case. If you think your type might be commonly used, you can also open an issue to ask for an addon library to provide for that type.

### Gson support

The `krate-gson` artifact provides you a `gsonPref` delegate which can store any arbitrary type, as long as GSON can serialize and deserialize it. This addon, like the base library, is available from `mavenCentral()`:

```groovy
implementation 'hu.autsoft:krate-gson:0.4.0'
```

Its basic usage is the same as with any of the base library's delegates:

```kotlin
class GsonKrate(context: Context) : SimpleKrate(context) {
    var user: User? by gsonPref("user")
    var savedArticles: List<Article>? by gsonPref("articles")
}
```

By default, the `Gson` instance created by a simple `Gson()` constructor call is used. If you want to provide your own `Gson` instance that you've configured, you can set the `gson` extension property on your Krate. Any `gsonPref` delegates within this Krate will use this instance for serialization and deserialization.

```kotlin
class CustomGsonKrate(context: Context) : SimpleKrate(context) {
    init {
        gson = GsonBuilder().create()
    }
}
``` 

### Moshi support 

Similarly to [krate-gson], the following artifacts add support for using Moshi to serialize values to `SharedPreferences` via Krate.

Since Moshi supports both reflection-based serialization and code generation, there are multiple Krate artifacts for different use cases. You should always include only  one of the dependencies below, otherwise you'll end up with a dexing error.

The usage of the Krate integration is the same for both setups:

```kotlin
class MoshiKrate(context: Context) : SimpleKrate(context) {
    var user: User? by moshiPref("user")
    var savedArticles: List<Article>? by moshiPref("articles")
}
```

If you want to provide your own `Moshi` instance that you've configured with your own custom adapters, you can set the `moshi` extension property on your Krate. Any `moshiPref` delegates within this Krate will use this instance for serialization and deserialization.

```kotlin
class CustomMoshiKrate(context: Context) : SimpleKrate(context) {
    init {
        moshi = Moshi.Builder().build()
    }
}
```

#### Only codegen

If you only want to use Moshi adapters that you generate via Moshi's [codegen facilities](https://github.com/square/moshi#codegen), you can use the following Krate artifact in your project to make use of these adapters: 

```groovy
implementation 'hu.autsoft:krate-moshi-codegen:0.4.0'
```

This will give you a default `Moshi` instance created by a call to `Moshi.Builder().build()`. This instance will find and use any of the adapters generated by Moshi's codegen automatically.

#### Only reflection, or mixed use of reflection and codegen

If you rely on [reflection](https://github.com/square/moshi#reflection) for your Moshi serialization, and therefore need a `KotlinJsonAdapterFactory` included in your `Moshi` instance, use the following Krate Moshi dependency:

```groovy
implementation 'hu.autsoft:krate-moshi-reflect:0.4.0'
```

The default `Moshi` instance from this dependency will include the aforementioned factory, and be able to serialize any Kotlin class. Note that this approach relies on the `kotlin-reflect` library, which is a large dependency.

You may choose to use Moshi's codegen for some classes in your project, and serialize the ones with no adapters generated with the default approach via reflection. For this mixed use case, you should probably choose this dependency.

# License

```
Copyright 2020 AutSoft

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
