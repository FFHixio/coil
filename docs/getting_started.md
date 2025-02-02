# Getting Started

## Artifacts

Coil has 8 artifacts published to `mavenCentral()`:

* `io.coil-kt:coil`: The default artifact which depends on `io.coil-kt:coil-base`, creates a singleton `ImageLoader`, and includes the `ImageView` extension functions.
* `io.coil-kt:coil-base`: A subset of `io.coil-kt:coil` which **does not** include the singleton `ImageLoader` and the `ImageView` extension functions.
* `io.coil-kt:coil-compose`: Includes support for [Jetpack Compose](https://developer.android.com/jetpack/compose).
* `io.coil-kt:coil-compose-base`: A subset of `io.coil-kt:coil-compose` which does not include functions that depend on the singleton `ImageLoader`.
* `io.coil-kt:coil-gif`: Includes two [decoders](../api/coil-base/coil.decode/-decoder) to support decoding GIFs. See [GIFs](gifs.md) for more details.
* `io.coil-kt:coil-svg`: Includes a [decoder](../api/coil-base/coil.decode/-decoder) to support decoding SVGs. See [SVGs](svgs.md) for more details.
* `io.coil-kt:coil-video`: Includes a [decoder](../api/coil-base/coil.decode/-decoder) to support decoding frames from [any of Android's supported video formats](https://developer.android.com/guide/topics/media/media-formats#video-codecs). See [videos](videos.md) for more details.
* `io.coil-kt:coil-bom`: Includes a [bill of materials](https://docs.gradle.org/7.2/userguide/platforms.html#sub:bom_import). Importing `coil-bom` allows you to depend on other Coil artifacts without specifying a version.

## Java 8

Coil requires [Java 8 bytecode](https://developer.android.com/studio/write/java8-support). To enable this add the following to your Gradle build script:

Gradle (`.gradle`):

```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
        freeCompilerArgs += "-Xjvm-default=all" // Only required for 2.x.
    }
}
```

Gradle Kotlin DSL (`.gradle.kts`):

```kotlin
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
        freeCompilerArgs += "-Xjvm-default=all" // Only required for 2.x.
    }
}
```

## Image Loaders

[`ImageLoader`](image_loaders.md)s are service classes that execute [`ImageRequest`](image_requests.md)s. `ImageLoader`s handle caching, data fetching, image decoding, request management, bitmap pooling, memory management, and more. New instances can be created and configured using a builder:

```kotlin
val imageLoader = ImageLoader.Builder(context)
    .availableMemoryPercentage(0.25)
    .crossfade(true)
    .build()
```

Coil performs best when you create a single `ImageLoader` and share it throughout your app. This is because each `ImageLoader` has its own memory cache, bitmap pool, and network observer.

It's recommended, though not required, to call [`shutdown`](../api/coil-base/coil/-image-loader/shutdown.html) when you've finished using an image loader. Calling `shutdown` preemptively frees its memory and cleans up any observers. If you only create and use a single `ImageLoader`, you do not need to shut it down as it will be freed when your app is killed.

## Image Requests

[`ImageRequest`](image_requests.md)s are value classes that are executed by [`ImageLoader`](image_loaders.md)s. They describe where an image should be loaded from, how it should be loaded, and any extra parameters. An `ImageLoader` has two methods that can execute a request:

- `enqueue`: Enqueues the `ImageRequest` to be executed asynchronously on a background thread.
- `execute`: Executes the `ImageRequest` in the current coroutine and returns an [`ImageResult`](../api/coil-base/coil.request/-image-result).

All requests should set `data` (i.e. url, uri, file, drawable resource, etc.). This is what the `ImageLoader` will use to decide where to fetch the image data from. If you do not set `data`, it will default to [`NullRequestData`](../api/coil-base/coil.request/-null-request-data).

Additionally, you likely want to set a `target` when enqueuing a request. It's optional, but the `target` is what will receive the loaded placeholder/success/error drawables. Executed requests return an `ImageResult` which has the success/error drawable.

Here's an example:

```kotlin
// enqueue
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    .target(imageView)
    .build()
val disposable = imageLoader.enqueue(request)

// execute
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    .build()
val result = imageLoader.execute(request)
```

## Singleton

If you are using the `io.coil-kt:coil` artifact, you can set the singleton [`ImageLoader`](image_loaders.md) instance by either:

- Implementing `ImageLoaderFactory` on your `Application` class (prefer this method):

```kotlin
class MyApplication : Application(), ImageLoaderFactory {

    override fun newImageLoader(): ImageLoader {
        return ImageLoader.Builder(applicationContext)
            .crossfade(true)
            .okHttpClient {
                OkHttpClient.Builder()
                    .cache(CoilUtils.createDefaultCache(applicationContext))
                    .build()
            }
            .build()
    }
}
```

- **Or** calling `Coil.setImageLoader`:

```kotlin
val imageLoader = ImageLoader.Builder(context)
    .crossfade(true)
    .okHttpClient {
        OkHttpClient.Builder()
            .cache(CoilUtils.createDefaultCache(context))
            .build()
    }
    .build()
Coil.setImageLoader(imageLoader)
```

The singleton `ImageLoader` can be retrieved using the `Context.imageLoader` extension function:

```kotlin
val imageLoader = context.imageLoader
```

Setting the singleton `ImageLoader` is optional. If you don't set one, Coil will lazily create an `ImageLoader` with the default values.

If you're using the `io.coil-kt:coil-base` artifact, you should create your own `ImageLoader` instance(s) and inject them throughout your app with dependency injection. [Read more about dependency injection here](../image_loaders/#singleton-vs-dependency-injection).

!!! Note
    If you set a custom `OkHttpClient`, you must set a `cache` implementation or the `ImageLoader` will have no disk cache. A default Coil cache instance can be created using [`CoilUtils.createDefaultCache`](../api/coil-base/coil.util/-coil-utils/create-default-cache.html).

## ImageView Extension Functions

The `io.coil-kt:coil` artifact provides a set of type-safe `ImageView` extension functions. Here's an example for loading a URL into an `ImageView`:

```kotlin
imageView.load("https://www.example.com/image.jpg")
```

The above call is equivalent to:

```kotlin
val imageLoader = imageView.context.imageLoader
val request = ImageRequest.Builder(imageView.context)
    .data("https://www.example.com/image.jpg")
    .target(imageView)
    .build()
imageLoader.enqueue(request)
```

`ImageView.load` calls can be configured with an optional trailing lambda parameter:

```kotlin
imageView.load("https://www.example.com/image.jpg") {
    crossfade(true)
    placeholder(R.drawable.image)
    transformations(CircleCropTransformation())
}
```

See the docs [here](../api/coil-singleton/coil-singleton/coil/) for more information.

## Supported Data Types

The base data types that are supported by all `ImageLoader` instances are:

* String (mapped to a Uri)
* HttpUrl
* Uri (`android.resource`, `content`, `file`, `http`, and `https` schemes)
* File
* @DrawableRes Int
* Drawable
* Bitmap
* ByteBuffer

## Supported Image Formats

All `ImageLoader`s support the following non-animated file types:

* BMP
* JPEG
* PNG
* WebP
* HEIF (Android 8.0+)

Additionally, Coil has extension libraries for the following file types:

* `coil-gif`: GIF, animated WebP (Android 9.0+), animated HEIF (Android 11.0+)
* `coil-svg`: SVG
* `coil-video`: Static video frames from any [video codec supported by Android](https://developer.android.com/guide/topics/media/media-formats#video-codecs)

## Preloading

To preload an image into memory, enqueue or execute an `ImageRequest` without a `Target`:

```kotlin
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    // Optional, but setting a ViewSizeResolver will conserve memory by limiting the size the image should be preloaded into memory at.
    .size(ViewSizeResolver(imageView))
    .build()
imageLoader.enqueue(request)
```

To preload a network image only into the disk cache, disable the memory cache for the request:

```kotlin
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    .memoryCachePolicy(CachePolicy.DISABLED)
    .build()
imageLoader.enqueue(request)
```

## Cancelling Requests

`ImageRequest`s will be automatically cancelled in the following cases:

- `request.lifecycle` reaches the `DESTROYED` state.
- `request.target` is a `ViewTarget` and its `View` is detached.

Additionally, `ImageLoader.enqueue` returns a [Disposable](../api/coil-base/coil.request/-disposable/), which can be used to dispose the request (which cancels it and frees its associated resources):

```kotlin
val disposable = imageView.load("https://www.example.com/image.jpg")

// Cancel the request.
disposable.dispose()
```

## Memory Cache

Each `ImageLoader` has its own `MemoryCache` of recently loaded images. To read/write a `Bitmap` to the memory cache, you need a `MemoryCache.Key`. There are two ways to get a `MemoryCache.Key`:

- Create a `MemoryCache.Key` using its `String` constructor: `MemoryCache.Key("my_cache_key")`
- Get the `MemoryCache.Key` from an executed request:

```kotlin
// If using the ImageLoader singleton
val memoryCacheKey = imageView.metadata.memoryCacheKey

// Enqueue
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    .target(imageView)
    .listener { _, metadata ->
        val memoryCacheKey = metadata.memoryCacheKey
    }
    .build()
imageLoader.enqueue(request)

// Execute
val request = ImageRequest.Builder(context)
    .data("https://www.example.com/image.jpg")
    .build()
val result = imageLoader.execute(request) as SuccessResult
val memoryCacheKey = result.metadata.memoryCacheKey
```

Once you have the memory cache key, you can read/write to the memory cache synchronously:

```kotlin
// Get
val bitmap: Bitmap? = imageLoader.memoryCache[memoryCacheKey]

// Set
imageLoader.memoryCache[memoryCacheKey] = bitmap
```
