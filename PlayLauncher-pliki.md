# Play Launcher: pliki projektu

Tu jest cały kod launchera. Wszystko robisz w przeglądarce na github.com, bez instalowania programów.

## Jak to wgrać (krok po kroku)

1. Na github.com utwórz repozytorium (**+** → **New repository**, nazwa np. `play-launcher`, **Create repository**).
2. Dla **każdego pliku niżej**: w repozytorium kliknij **Add file → Create new file**. W polu nazwy wpisz dokładnie ścieżkę z nagłówka (np. `app/build.gradle.kts`). Kiedy wpisujesz ukośnik `/`, GitHub sam tworzy foldery. W dużym polu pod spodem wklej zawartość bloku kodu i kliknij **Commit changes**.
3. Na końcu wgraj plik **index.html** (osobny plik, który też pobrałeś) do folderu `app/src/main/assets/`. W tym celu kliknij **Add file → Create new file**, wpisz nazwę `app/src/main/assets/pusty.txt`, wpisz w środku cokolwiek (np. `x`) i zatwierdź. Dzięki temu folder istnieje. Potem wejdź w ten folder, kliknij **Add file → Upload files**, przeciągnij `index.html` i zatwierdź.
4. Zakładka **Actions** → **Build APK** → **Run workflow**. Po 3 do 6 minutach pobierz plik z **Artifacts** (PlayLauncher-apk), rozpakuj go i zainstaluj: `adb install -r app-debug.apk`.

Kopiuj tylko zawartość bloków kodu (bez linii z trzema apostrofami).

---

## Plik: `.github/workflows/build.yml`

```yaml
name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main, master]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"
      - run: gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: PlayLauncher-apk
          path: app/build/outputs/apk/debug/app-debug.apk
```

---

## Plik: `settings.gradle.kts`

```kotlin
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "PlayLauncher"
include(":app")
```

---

## Plik: `build.gradle.kts`

```kotlin
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
```

---

## Plik: `gradle.properties`

```properties
org.gradle.jvmargs=-Xmx2g
kotlin.code.style=official
```

---

## Plik: `app/build.gradle.kts`

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}
android {
    namespace = "pl.launcher.play"
    compileSdk = 34
    defaultConfig {
        applicationId = "pl.launcher.play"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}
```

---

## Plik: `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-feature android:name="android.software.leanback" android:required="false" />
    <uses-feature android:name="android.hardware.touchscreen" android:required="false" />
    <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" />
    <application
        android:allowBackup="false"
        android:label="Play Launcher"
        android:icon="@drawable/banner"
        android:banner="@drawable/banner"
        android:theme="@android:style/Theme.Material.NoActionBar">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTask"
            android:configChanges="orientation|screenSize|keyboardHidden|keyboard|navigation">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.HOME" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## Plik: `app/src/main/res/drawable/banner.xml`

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="320dp" android:height="180dp"
    android:viewportWidth="320" android:viewportHeight="180">
    <path android:fillColor="#1D2B53" android:pathData="M0,0h320v180h-320z" />
    <path android:fillColor="#FFFFFF" android:pathData="M135,60L200,90L135,120Z" />
</vector>
```

---

## Plik: `app/src/main/java/pl/launcher/play/MainActivity.kt`

```kotlin
package pl.launcher.play

import android.app.Activity
import android.content.Intent
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.drawable.Drawable
import android.os.Bundle
import android.util.Base64
import android.view.View
import android.webkit.JavascriptInterface
import android.webkit.WebView
import android.webkit.WebViewClient
import org.json.JSONArray
import org.json.JSONObject
import java.io.ByteArrayOutputStream

class MainActivity : Activity() {
    private lateinit var web: WebView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        web = WebView(this)
        web.setBackgroundColor(0xFF0B1020.toInt())
        val s = web.settings
        s.javaScriptEnabled = true
        s.domStorageEnabled = true
        s.allowFileAccess = true
        s.mediaPlaybackRequiresUserGesture = false
        web.webViewClient = WebViewClient()
        web.addJavascriptInterface(Bridge(), "Android")
        setContentView(web)
        web.loadUrl("file:///android_asset/index.html")
        web.requestFocus()
        hideBars()
    }

    private fun hideBars() {
        window.decorView.systemUiVisibility = (View.SYSTEM_UI_FLAG_FULLSCREEN
                or View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                or View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
                or View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                or View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION)
    }

    override fun onWindowFocusChanged(hasFocus: Boolean) {
        super.onWindowFocusChanged(hasFocus)
        if (hasFocus) hideBars()
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        sendEscape()
    }

    // Launcher nie powinien sie zamykac; Wstecz zamyka nakladki w stronie.
    @Deprecated("Deprecated in Java")
    override fun onBackPressed() {
        sendEscape()
    }

    private fun sendEscape() {
        web.evaluateJavascript(
            "document.dispatchEvent(new KeyboardEvent(\"keydown\",{key:\"Escape\",bubbles:true,cancelable:true}))",
            null
        )
    }

    inner class Bridge {
        @JavascriptInterface
        fun launch(pkg: String): Boolean {
            val pm = packageManager
            val i = pm.getLeanbackLaunchIntentForPackage(pkg)
                ?: pm.getLaunchIntentForPackage(pkg)
                ?: return false
            i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            return try {
                startActivity(i)
                true
            } catch (e: Exception) {
                false
            }
        }

        @JavascriptInterface
        fun listApps(): String {
            val pm = packageManager
            val leanback = Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LEANBACK_LAUNCHER)
            val normal = Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER)
            val seen = HashSet<String>()
            val arr = JSONArray()
            val all = pm.queryIntentActivities(leanback, 0) + pm.queryIntentActivities(normal, 0)
            for (ri in all) {
                val pkg = ri.activityInfo.packageName
                if (pkg == packageName || !seen.add(pkg)) continue
                val o = JSONObject()
                o.put("p", pkg)
                o.put("l", ri.loadLabel(pm).toString())
                o.put("i", iconB64(ri.loadIcon(pm)))
                arr.put(o)
            }
            return arr.toString()
        }
    }

    private fun iconB64(d: Drawable): String {
        val size = 96
        val bmp = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
        val c = Canvas(bmp)
        d.setBounds(0, 0, size, size)
        d.draw(c)
        val bo = ByteArrayOutputStream()
        bmp.compress(Bitmap.CompressFormat.PNG, 100, bo)
        return Base64.encodeToString(bo.toByteArray(), Base64.NO_WRAP)
    }
}
```
