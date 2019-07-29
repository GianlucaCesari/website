---
title: Adding a Flutter Fragment to an Android app
short-title: Add a Flutter Fragment
description: Learn how to add a Flutter Fragment to your existing Android app.
---

{% asset
development/platform-integration/add-to-app-android/add-flutter-fragment/add-flutter-fragment_header.png
class="mw-100" alt="Add Flutter Fragment Header" %}

This guide describes how to add a Flutter `Fragment` to an existing Android app.
In Android, a [`Fragment`] represents a modular piece of a larger UI. A
`Fragment` might be used to present a sliding drawer, tabbed content, a page in
a `ViewPager`, or it might simply represent a normal screen in a
single-`Activity` app. Flutter provides a `FlutterFragment` so that developers
can present a Flutter experience any place that they can use a regular
`Fragment`.

[`Fragment`]: https://developer.android.com/guide/components/fragments

If an `Activity` is equally applicable for your application needs, consider
[using a `FlutterActivity`] instead of a `FlutterFragment`, which is quicker and
easier to use.

[using a `FlutterActivity`]: /docs/development/platform-integration/add-to-app-android/add-flutter-screen

`FlutterFragment` allows developers to control the following details of the
Flutter experience within the `Fragment`:

 * Initial Flutter route
 * Dart entrypoint to execute
 * Opaque vs translucent background
 * Whether `FlutterFragment` should control its surrounding `Activity`

`FlutterFragment` also comes with a number of calls that must be forwarded from
its surrounding `Activity`. These calls allow Flutter to react appropriately to
OS events.

All varieties of `FlutterFragment`, and its requirements, are described in this
guide.

## Add a `FlutterFragment` to an `Activity`

The first thing to do to use a `FlutterFragment` is to add it to a host
`Activity`.

To add a `FlutterFragment` to a host `Activity`, start by instantiating and
attaching an instance of `FlutterFragment` in `onCreate()` within the
`Activity`, or at some other time that works for your app:

```java
public class MyActivity extends FragmentActivity {
    // Define a tag String to represent the FlutterFragment within this
    // Activity's FragmentManager. This value can be whatever you'd like.
    private static final String TAG_FLUTTER_FRAGMENT = "flutter_fragment";

    // Declare a local variable to reference the FlutterFragment so that you
    // can forward calls to it later.
    private FlutterFragment flutterFragment;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Inflate a layout that has a container for your FlutterFragment. For
        // this example, assume that a FrameLayout exists with an ID of 
        // R.id.fragment_container.
        setContentView(R.layout.my_activity_layout);

        // Get a reference to the Activity's FragmentManager to add a new 
        // FlutterFragment, or find an existing one.
        FragmentManager fragmentManager = getSupportFragmentManager();

        // Attempt to find an existing FlutterFragment, in case this is not the
        // first time that onCreate() has run.
        flutterFragment = (FlutterFragment) fragmentManager
            .findFragmentByTag(TAG_FLUTTER_FRAGMENT);

        // Create and attach a FlutterFragment if one does not yet exist.
        if (flutterFragment == null) {
            flutterFragment = new FlutterFragment.Builder().build();
            
            fragmentManager
                .beginTransaction()
                .add(
                    R.id.fragment_container, 
                    flutterFragment, 
                    TAG_FLUTTER_FRAGMENT
                )
                .commit();
        }
    }
}
```

The above code is sufficient to render a Flutter UI that begins with a call to
your `main()` Dart entrypoint and an initial Flutter route of `/`. However, this
code is not sufficient to achieve all expected Flutter behavior. Flutter depends
on various OS signals that need to be forwarded from your host `Activity` to
`FlutterFragment`. These calls are shown below:

```java
public class MyActivity extends FragmentActivity {
    @Override
    public void onPostResume() {
        super.onPostResume();
        flutterFragment.onPostResume();
    }

    @Override
    protected void onNewIntent(@NonNull Intent intent) {
        flutterFragment.onNewIntent(intent);
    }

    @Override
    public void onBackPressed() {
        flutterFragment.onBackPressed();
    }

    @Override
    public void onRequestPermissionsResult(
        int requestCode, 
        @NonNull String[] permissions, 
        @NonNull int[] grantResults
    ) {
        flutterFragment.onRequestPermissionsResult(
            requestCode, 
            permissions, 
            grantResults
        );
    }

    @Override
    public void onUserLeaveHint() {
        flutterFragment.onUserLeaveHint();
    }

    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        flutterFragment.onTrimMemory(level);
    }
}
```

With the above OS signals forwarded to Flutter, your `FlutterFragment` works as
expected. You have now added a `FlutterFragment` to your existing Android app.

## Using a pre-warmed `FlutterEngine`

By default, a `FlutterFragment` creates its own instance of a `FlutterEngine`,
which requires non-trivial warmup time. This means your user sees a blank
`Fragment` for a brief moment. You can mitigate most of this warmup time by
using an existing, pre-warmed instance of `FlutterEngine`.

Please see the [instructions for instantiating and starting a `FlutterEngine`].

[instructions for instantiating and starting a `FlutterEngine`]: /docs/development/platform-integrations/add-to-app-android/flutter-engine

To use a pre-warmed `FlutterEngine` in a `FlutterFragment`, subclass
`FlutterFragment` and override `provideFlutterEngine()` as shown below:

```java
public class MyFlutterFragment extends FlutterFragment {
    @Override
    protected FlutterEngine provideFlutterEngine(@NonNull Context context) {
        // Obtain a reference to, and return, your pre-warmed FlutterEngine.
        // For example:
        return ((MyApplication) context.getApplication())
            .getPreWarmedFlutterEngine();
    }

    // When providing a pre-warmed engine within a FlutterFragment subclass,
    // be sure to override this method and return "true" so that this
    // FlutterFragment does not destroy your FlutterEngine when the Fragment
    // is destroyed.
    @Override
    protected boolean retainFlutterEngineAfterFragmentDestruction() {
        return true;
    }
}
```

By providing a pre-warmed `FlutterEngine` as shown above, your app renders the
first Flutter frame as quickly as possible.

## Display a splash screen

The initial display of Flutter content requires some amount of time, even if a
pre-warmed `FlutterEngine` is used. To help improve the user experience around
this brief waiting period, Flutter supports the display of a splash screen until
Flutter renders its first frame. For instructions about showing a splash screen,
please see the [Android splash screen guide].

[Android splash screen guide]: /docs/development/platform-integration/add-to-app-android/add-splash-screen

## Run Flutter with a specified initial route

An Android app might contain many independent Flutter experiences, running in
different `FlutterFragment`s, with different `FlutterEngine`s. In these
scenarios, it is common for each Flutter experience to begin with different
initial routes (routes other than `/`). To facilitate this, `FlutterFragment`'s
`Builder` allows you to specify a desired initial route, as shown below:

```java
FlutterFragment flutterFragment = new FlutterFragment.Builder()
    .initialRoute("myInitialRoute/")
    .build();
```

{{site.alert.note}} 
  `FlutterFragment`'s initial route property has no effect when a pre-warmed
  `FlutterEngine` is used because the pre-warmed `FlutterEngine` has already
  chosen an initial route. The initial route can be chosen explicitly when
  pre-warming a `FlutterEngine`.
{{site.alert.end}}

## Run Flutter from a specified entrypoint

Similar to varying initial routes, different `FlutterFragments` may wish to
execute different Dart entrypoints. In a typical Flutter app there is only one
Dart entrypoint: `main()`, but you can define other entrypoints. See [defining
Dart entrypoints] for instructions on defining such entrypoints.

[defining Dart entrypoints]: /docs/development/platform-integration/add-to-app-android/custom-dart-entrypoints

`FlutterFragment` supports specification of the desired Dart entrypoint to
execute for the given Flutter experience. To specify an entrypoint, build
`FlutterFragment` as follows:

```java
FlutterFragment flutterFragment = new FlutterFragment.Builder()
    .dartEntrypoint("mySpecialEntrypoint")
    .build();
```

The above `FlutterFragment` configuration results in the execution of a Dart
entrypoint called `mySpecialEntrypoint()`. Notice that the parentheses `()` are
not included in the `dartEntrypoint` `String` name.

{{site.alert.note}} 
  `FlutterFragment`'s Dart entrypoint property has no effect when a pre-warmed
  `FlutterEngine` is used because the pre-warmed `FlutterEngine` has already
  executed a Dart entrypoint. The Dart entrypoint can be chosen explicitly when
  pre-warming a `FlutterEngine`.
{{site.alert.end}}

## Control `FlutterFragment`'s render mode

`FlutterFragment` can either use a `SurfaceView` to render its Flutter content,
or it can use a `TextureView`. The default is `SurfaceView`, which is
significantly better for performance than `TextureView`. However, `SurfaceView`
cannot be interleaved in the middle of an Android `View` hierarchy. A
`SurfaceView` must either be the bottom-most `View` in the hierarchy, or the
top-most `View` in the hierarchy. Additionally, on Android versions before
Android N, `SurfaceView`s cannot be animated becuase their layout and rendering
are not synchronized with the rest of the `View` hierarchy. If either of these
use-cases are requirements for your app, you need to use `TextureView` instead
of `SurfaceView`. Select a `TextureView` by building a `FlutterFragment` with a
`texture` `RenderMode`:

```java
FlutterFragment flutterFragment = new FlutterFragment.Builder()
    .renderMode(FlutterView.RenderMode.texture)
    .build();
```

Using the configuration above, the resulting `FlutterFragment` renders its UI to
a `TextureView`.

## Display a `FlutterFragment` with transparency

By default, `FlutterFragment` renders with an opaque background, using a
`SurfaceView` (see the section above titled "Control `FlutterFragment`'s render
mode"). That background is black for any pixels that are not painted by Flutter.
Rendering with an opaque background is the preferred rendering mode for
performance reasons. Flutter rendering with transparency on Android negatively
impacts performance. However, there are many designs that require transparent
pixels in the Flutter experience that show through to the underlying Android UI.
For this reason, Flutter supports translucency in a `FlutterFragment`.

{{site.alert.note}} 
  Both `SurfaceView` and `TextureView` support transparency. However, when a
  `SurfaceView` is instructed to render with transparency, it positions itself
  at a higher z-index than all other Android `View`s, which means it appears
  above all other `View`s. This is a limitation of `SurfaceView`, itself. If it
  is acceptable to render your Flutter experience on top of all other content,
  then `FlutterFragment`'s default `RenderMode` of `surface` is the `RenderMode`
  that you should use. However, if you need to display Android `View`s both
  below and above your Flutter experience, then you must specify a
  `RenderMode` of `texture`. See the previous section titled "Control
  `FlutterFragment`'s render mode" for information on controlling the
  `RenderMode`. 
{{site.alert.end}}

To enable transparency for a `FlutterFragment`, build it with the following
configuration:

```java
FlutterFragment flutterFragment = new FlutterFragment.Builder()
    .transparencyMode(FlutterView.TransparencyMode.transparent)
    .build();
```

## The relationship beween `FlutterFragment` and its `Activity`

Some apps choose to use `Fragment`s as entire Android screens. In these apps, it
would be reasonable for a `Fragment` to control system chrome like Android's
status bar, navigation bar, and orientation.

{% asset
development/platform-integration/add-to-app-android/add-flutter-fragment/add-flutter-fragment_fullscreen.png
class="mw-100" alt="Fullscreen Flutter" %}

In other apps, `Fragment`s are used to represent only a portion of a UI. A
`FlutterFragment` might be used to implement the inside of a drawer, or a video
player, or a single card. In these situations, it would be inappropriate for the
`FlutterFragment` to affect Android's system chrome because there are other UI
pieces within the same `Window`.

{% asset
development/platform-integration/add-to-app-android/add-flutter-fragment/add-flutter-fragment_partial-ui.png
class="mw-100" alt="Flutter as Partial UI" %}

`FlutterFragment` comes with a concept that helps differentiate between the case
where a `FlutterFragment` should be able to control its host `Activity`, and the
cases where a `FlutterFragment` should only affect its own behavior. To prevent
a `FlutterFragment` from exposing its `Activity` to Flutter plugins, and to 
prevent Flutter from controlling the `Activity`'s system UI, use the
`shouldAttachEngineToActivity()` method in `FlutterFragment`'s `Builder` as
shown below.

```java
FlutterFragment flutterFragment = new FlutterFragment.Builder()
    .shouldAttachEngineToActivity(false)
    .build();
```

Passing `false` to the `shouldAttachEngineToActivity()` `Builder` method
prevents Flutter from interacting with the surrounding `Activity`. The default
value is `true`, which allows Flutter and Flutter plugins to interact with
surrounding `Activity`.

{{site.alert.note}} 
  Some plugins may expect or require an `Activity` reference. Ensure that none 
  of your plugins require an `Activity` before disabling access.
{{site.alert.end}}