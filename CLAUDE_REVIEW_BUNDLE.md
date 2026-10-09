# Radin Assistant — Claude Technical Review Bundle
Generated: 2026-10-09 (project timezone Iran, approximate)
Purpose: independent review of Android Accessibility service detection/readiness.
Source branch: `diag/accessibility-detect`
Source commit: `e60bb4b95fd43f750db08ab9f5b0b70a24fc7c91`
This is a diagnostic snapshot only. Do not merge, sign, install, or modify the production APK based on review alone.

## Evidence status
- GitHub Actions run: https://github.com/younesshiva-ai/Radinassistant-/actions/runs/37883653890
- Run ID: `37883653890`; conclusion: success.
- Confirmed log: `BUILD SUCCESSFUL in 2m 37s`; `:app:assembleDebug` compiled the branch.
- Artifact: `radin-accessibility-diagnostic-unsigned`, artifact ID `11595765668`, ZIP size 15,147 bytes, reported digest `sha256:afefa82c7535339d73ffdda485a6bac93291baeecb3ea0141f1a9af98f30503c`.
- Artifact has not been installed or tested on the target phone. Actual APK signing status has not been directly checked; the workflow omits the project's custom signing environment, but default debug signing may still occur.
- Target device context reported by project owner: Redmi 14C, Android 15, HyperOS 3.
- The `PermissionManager` change is plausible but the root cause is not established without device behavior/logs.
- Build warnings observed: manifest `package=` is ignored for namespace by AGP; `RadinAccessibilityService.java` uses a deprecated API. Neither stopped the build.
- No credentials, signing keys, Git history, APK, or CI secrets are intentionally included in this review bundle. The Gradle file only refers to environment variable names; actual secret values are not present.

## Questions for independent review
Please inspect only the supplied source and evidence; do not assume unprovided device behavior.
1. Is `Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES` parsed robustly here? Identify Android-specific edge cases, component normalization, casing, and null behavior.
2. Is `new ComponentName(context, RadinAccessibilityService.class)` correct for comparing entries from the enabled-services setting?
3. Does `isServiceInstanceAvailable()` plus the static singleton in `onServiceConnected/onUnbind` reliably represent service readiness? Identify lifecycle or race conditions.
4. Inspect the Manifest and XML metadata for any evidence-based service registration/configuration issue. Clearly distinguish definite errors from hypotheses; do not recommend changing `android:exported` without concrete reasoning.
5. Trace the button state and refresh flow in `MainActivity`. What exact instrumentation or minimal device test would distinguish “enabled in Settings” from “service connected”?
6. Review resource recycling and node traversal in `RadinAccessibilityService` only for issues relevant to this diagnostic path.
7. Propose the smallest safe next step, with confidence level and rationale. Do not suggest merging to `main` or replacing the known-good APK before device verification.
8. List the precise logs / outputs needed if the source alone cannot prove the root cause.

Please return: (A) definite findings, (B) plausible hypotheses, (C) unknowns requiring device evidence, (D) minimal next step, (E) risks of proposed changes.

## File: `app/src/main/java/com/radin/assistant/control/PermissionManager.java`
Source blob SHA: `87c8a595ffce746021075f6d90243631ca9adcab`

```java
package com.radin.assistant.control;

import android.accessibilityservice.AccessibilityService;
import android.content.ComponentName;
import android.content.Context;
import android.provider.Settings;
import android.text.TextUtils;

public final class PermissionManager {
    private PermissionManager() {}

    public static boolean isAccessibilityEnabled(Context context) {
        String enabledServices = Settings.Secure.getString(
                context.getContentResolver(),
                Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES
        );
        if (TextUtils.isEmpty(enabledServices)) {
            return false;
        }

        ComponentName expected =
                new ComponentName(context, RadinAccessibilityService.class);

        TextUtils.SimpleStringSplitter splitter =
                new TextUtils.SimpleStringSplitter(':');
        splitter.setString(enabledServices);
        while (splitter.hasNext()) {
            ComponentName enabled =
                    ComponentName.unflattenFromString(splitter.next());
            if (expected.equals(enabled)) {
                return true;
            }
        }
        return false;
    }

    public static boolean isServiceInstanceAvailable() {
        return RadinAccessibilityService.getInstance() != null;
    }
}
```


## File: `app/src/main/java/com/radin/assistant/control/RadinAccessibilityService.java`
Source blob SHA: `ab00c4c7122ec224143111a52d684c1a9dd6d12a`

```java
package com.radin.assistant.control;

import android.accessibilityservice.AccessibilityService;
import android.accessibilityservice.GestureDescription;
import android.graphics.Path;
import android.os.Build;
import android.os.Bundle;
import android.view.accessibility.AccessibilityEvent;
import android.view.accessibility.AccessibilityNodeInfo;

public class RadinAccessibilityService extends AccessibilityService {
    private static RadinAccessibilityService instance;

    public static RadinAccessibilityService getInstance() {
        return instance;
    }

    @Override
    protected void onServiceConnected() {
        super.onServiceConnected();
        instance = this;
        AuditLogger.record("accessibility.service_connected", true);
    }

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        // Event stream is intentionally passive; commands are executed explicitly.
    }

    @Override
    public void onInterrupt() {
        // Required by AccessibilityService.
    }

    public boolean performBack() {
        boolean ok = performGlobalAction(GLOBAL_ACTION_BACK);
        AuditLogger.record("ui.back", ok);
        return ok;
    }

    public boolean performHome() {
        boolean ok = performGlobalAction(GLOBAL_ACTION_HOME);
        AuditLogger.record("ui.home", ok);
        return ok;
    }

    public boolean clickText(String text) {
        AccessibilityNodeInfo root = getRootInActiveWindow();
        AccessibilityNodeInfo node = findText(root, text);
        boolean ok = node != null && node.isClickable()
                && node.performAction(AccessibilityNodeInfo.ACTION_CLICK);
        AuditLogger.record("ui.click", ok);
        if (node != null) node.recycle();
        return ok;
    }

    public boolean longClickText(String text) {
        AccessibilityNodeInfo root = getRootInActiveWindow();
        AccessibilityNodeInfo node = findText(root, text);
        boolean ok = node != null && node.isLongClickable()
                && node.performAction(AccessibilityNodeInfo.ACTION_LONG_CLICK);
        AuditLogger.record("ui.long_click", ok);
        if (node != null) node.recycle();
        return ok;
    }

    public boolean typeText(String text) {
        AccessibilityNodeInfo root = getRootInActiveWindow();
        AccessibilityNodeInfo node = findEditable(root);
        boolean ok = false;
        if (node != null) {
            Bundle args = new Bundle();
            args.putCharSequence(
                    AccessibilityNodeInfo.ACTION_ARGUMENT_SET_TEXT_CHARSEQUENCE, text);
            ok = node.performAction(AccessibilityNodeInfo.ACTION_SET_TEXT, args);
            node.recycle();
        }
        AuditLogger.record("ui.type", ok);
        return ok;
    }

    public boolean swipe(float x1, float y1, float x2, float y2, long durationMs) {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
            AuditLogger.record("ui.swipe", false);
            return false;
        }
        Path path = new Path();
        path.moveTo(x1, y1);
        path.lineTo(x2, y2);
        GestureDescription.StrokeDescription stroke =
                new GestureDescription.StrokeDescription(
                        path, 0, Math.max(1L, durationMs));
        boolean ok = dispatchGesture(
                new GestureDescription.Builder().addStroke(stroke).build(), null, null);
        AuditLogger.record("ui.swipe", ok);
        return ok;
    }

    private AccessibilityNodeInfo findText(AccessibilityNodeInfo node, String text) {
        if (node == null || text == null) return null;
        java.util.List<AccessibilityNodeInfo> matches =
                node.findAccessibilityNodeInfosByText(text);
        return matches != null && !matches.isEmpty() ? matches.get(0) : null;
    }

    private AccessibilityNodeInfo findEditable(AccessibilityNodeInfo node) {
        if (node == null) return null;
        if (node.isEditable()) return AccessibilityNodeInfo.obtain(node);
        for (int i = 0; i < node.getChildCount(); i++) {
            AccessibilityNodeInfo child = node.getChild(i);
            AccessibilityNodeInfo found = findEditable(child);
            if (child != null) child.recycle();
            if (found != null) return found;
        }
        return null;
    }

    @Override
    public boolean onUnbind(android.content.Intent intent) {
        instance = null;
        return super.onUnbind(intent);
    }
}
```


## File: `app/src/main/java/com/radin/assistant/control/RadinControlManager.java`
Source blob SHA: `df625807288fe0a7256e94948d70a40b5d96af16`

```java
package com.radin.assistant.control;

import android.content.Context;

public final class RadinControlManager {
    private final Context context;
    private final CommandPolicy policy;

    public RadinControlManager(Context context) {
        this.context = context.getApplicationContext();
        this.policy = new CommandPolicy();
    }

    public boolean canExecute(String action) {
        boolean allowed = policy.isAllowed(action);
        if (!allowed) {
            AuditLogger.recordDenied(action);
        }
        return allowed;
    }

    public boolean isAccessibilityReady() {
        return PermissionManager.isAccessibilityEnabled(context)
                && PermissionManager.isServiceInstanceAvailable();
    }

    public boolean back() {
        return execute("ui.back") && service().performBack();
    }

    public boolean home() {
        return execute("ui.home") && service().performHome();
    }

    public boolean clickText(String text) {
        return execute("ui.click") && service().clickText(text);
    }

    public boolean longClickText(String text) {
        return execute("ui.long_click") && service().longClickText(text);
    }

    public boolean typeText(String text) {
        return execute("ui.type") && service().typeText(text);
    }

    public boolean swipe(float x1, float y1, float x2, float y2, long durationMs) {
        return execute("ui.swipe")
                && service().swipe(x1, y1, x2, y2, durationMs);
    }

    private boolean execute(String action) {
        if (!canExecute(action)) return false;
        if (!isAccessibilityReady()) {
            AuditLogger.recordDenied(action + ".accessibility_not_ready");
            return false;
        }
        return true;
    }

    private RadinAccessibilityService service() {
        return RadinAccessibilityService.getInstance();
    }

    public CommandPolicy getPolicy() {
        return policy;
    }
}
```


## File: `app/src/main/java/com/radin/assistant/control/AuditLogger.java`
Source blob SHA: `68bbd2f21dbfc76447b59d9551ef24af404032b1`

```java
package com.radin.assistant.control;

import android.util.Log;

public final class AuditLogger {
    private static final String TAG = "RadinControl";

    private AuditLogger() {}

    public static void record(String action, boolean success) {
        Log.i(TAG, "action=" + action + ", success=" + success);
    }

    public static void recordDenied(String action) {
        Log.w(TAG, "action=" + action + ", denied=true");
    }
}
```


## File: `app/src/main/java/com/radin/assistant/control/CommandPolicy.java`
Source blob SHA: `b32864e3477d0f0bac1b45cd859df27c2dd59ee6`

```java
package com.radin.assistant.control;

import java.util.Arrays;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

public final class CommandPolicy {
    private final Set<String> allowedActions = Collections.unmodifiableSet(new HashSet<>(Arrays.asList(
            "device.status",
            "device.battery",
            "screen.read",
            "ui.find_text",
            "ui.click",
            "ui.long_click",
            "ui.swipe",
            "ui.scroll",
            "ui.back",
            "ui.home",
            "ui.type"
    )));

    public boolean isAllowed(String action) {
        return action != null && allowedActions.contains(action);
    }

    public Set<String> getAllowedActions() {
        return allowedActions;
    }
}
```


## File: `app/src/main/java/com/radin/assistant/MainActivity.java`
Source blob SHA: `5039ba390254824f26784564aa65aeb450ee3fb7`

```java
package com.radin.assistant;

import android.Manifest;
import android.app.Activity;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.provider.Settings;
import android.speech.RecognizerIntent;
import android.view.Gravity;
import android.view.View;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;

import com.radin.assistant.control.PermissionManager;
import com.radin.assistant.control.RadinControlManager;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.Locale;

public class MainActivity extends Activity {

    private static final int REQUEST_RECORD_AUDIO = 100;
    private static final int REQUEST_VOICE = 200;

    private TextView resultText;
    private Button accessibilityButton;
    private RadinControlManager controlManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        controlManager = new RadinControlManager(this);

        LinearLayout layout = new LinearLayout(this);
        layout.setOrientation(LinearLayout.VERTICAL);
        layout.setGravity(Gravity.CENTER);
        layout.setPadding(40, 40, 40, 40);

        TextView title = new TextView(this);
        title.setText("دستیار رادین");
        title.setTextSize(30);
        title.setGravity(Gravity.CENTER);

        TextView subtitle = new TextView(this);
        subtitle.setText("سلام داش ممد 👋\nرادین آماده‌ست.");
        subtitle.setTextSize(20);
        subtitle.setGravity(Gravity.CENTER);

        resultText = new TextView(this);
        resultText.setText("");
        resultText.setTextSize(20);
        resultText.setGravity(Gravity.CENTER);
        resultText.setPadding(10, 30, 10, 10);

        Button voiceButton = new Button(this);
        voiceButton.setText("🎤 صحبت با رادین");
        voiceButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startVoiceRecognition();
            }
        });

        accessibilityButton = new Button(this);
        accessibilityButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (PermissionManager.isAccessibilityEnabled(MainActivity.this)) {
                    resultText.setText(
                            controlManager.isAccessibilityReady()
                                    ? "✅ کنترل گوشی فعال و آماده است."
                                    : "✅ دسترسی کنترل گوشی فعال است؛ سرویس رادین در حال آماده‌شدن است."
                    );
                } else {
                    startActivity(new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS));
                }
            }
        });

        layout.addView(title);
        layout.addView(subtitle);

        LinearLayout.LayoutParams params =
                new LinearLayout.LayoutParams(
                        LinearLayout.LayoutParams.WRAP_CONTENT,
                        LinearLayout.LayoutParams.WRAP_CONTENT
                );
        params.topMargin = 40;
        layout.addView(voiceButton, params);

        LinearLayout.LayoutParams accessParams =
                new LinearLayout.LayoutParams(
                        LinearLayout.LayoutParams.WRAP_CONTENT,
                        LinearLayout.LayoutParams.WRAP_CONTENT
                );
        accessParams.topMargin = 16;
        layout.addView(accessibilityButton, accessParams);

        layout.addView(resultText);
        setContentView(layout);
        refreshAccessibilityButton();
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (accessibilityButton != null) {
            refreshAccessibilityButton();
        }
    }

    private void refreshAccessibilityButton() {
        if (PermissionManager.isAccessibilityEnabled(this)) {
            accessibilityButton.setText("♿ کنترل گوشی: فعال");
        } else {
            accessibilityButton.setText("♿ فعال‌سازی کنترل گوشی");
        }
    }

    private void startVoiceRecognition() {
        if (checkSelfPermission(Manifest.permission.RECORD_AUDIO)
                != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(
                    new String[]{Manifest.permission.RECORD_AUDIO},
                    REQUEST_RECORD_AUDIO
            );
            return;
        }

        if (!android.speech.SpeechRecognizer.isRecognitionAvailable(this)) {
            Toast.makeText(
                    this,
                    "تشخیص گفتار روی این گوشی در دسترس نیست",
                    Toast.LENGTH_LONG
            ).show();
            return;
        }

        Intent intent = new Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
        intent.putExtra(
                RecognizerIntent.EXTRA_LANGUAGE_MODEL,
                RecognizerIntent.LANGUAGE_MODEL_FREE_FORM
        );
        intent.putExtra(RecognizerIntent.EXTRA_LANGUAGE, "fa-IR");
        intent.putExtra(RecognizerIntent.EXTRA_PROMPT, "داش ممد، بگو...");
        startActivityForResult(intent, REQUEST_VOICE);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);

        if (requestCode != REQUEST_VOICE
                || resultCode != RESULT_OK
                || data == null) {
            return;
        }

        ArrayList<String> results =
                data.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS);

        if (results == null || results.isEmpty()) {
            return;
        }

        String spokenText = results.get(0).trim();
        handleVoiceCommand(spokenText);
    }

    private void handleVoiceCommand(String spokenText) {
        String normalized = spokenText
                .toLowerCase(Locale.ROOT)
                .replace('‌', ' ')
                .trim();

        if (normalized.contains("سلام رادین")) {
            resultText.setText(
                    "🗣️ " + spokenText + "\n\n🤖 سلام داش ممد 😎 خودم هستم!"
            );
            return;
        }

        if (normalized.contains("اسم من چیه")
                || normalized.contains("اسم من چیست")) {
            resultText.setText(
                    "🗣️ " + spokenText
                            + "\n\n🤖 اسم شما محمده، داش ممد 😎"
            );
            return;
        }

        if (normalized.contains("کنترل گوشی")
                || normalized.contains("فعال سازی کنترل")
                || normalized.contains("فعال‌سازی کنترل")) {
            if (PermissionManager.isAccessibilityEnabled(this)) {
                resultText.setText(
                        controlManager.isAccessibilityReady()
                                ? "🤖 کنترل رابط کاربری رادین فعال و آماده است."
                                : "✅ دسترسی کنترل گوشی فعال است؛ سرویس رادین در حال آماده‌شدن است."
                );
            } else {
                startActivity(new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS));
            }
            return;
        }

        if (normalized.contains("وضعیت کنترل")
                || normalized.contains("دسترسی کنترل")) {
            boolean ready = controlManager.isAccessibilityReady();
            resultText.setText(
                    ready
                            ? "🤖 کنترل رابط کاربری رادین فعال و آماده است."
                            : "⚠️ کنترل رابط کاربری هنوز فعال نشده؛ از تنظیمات Accessibility رادین را فعال کن."
            );
            return;
        }

        if (normalized.contains("ساعت چنده")
                || normalized.contains("ساعت چند است")
                || (normalized.contains("ساعت") && normalized.contains("چند"))) {
            SimpleDateFormat format =
                    new SimpleDateFormat("HH:mm", Locale.getDefault());
            resultText.setText(
                    "🗣️ " + spokenText
                            + "\n\n🤖 داش ممد، الان ساعت "
                            + format.format(new Date()) + " ـه 🕐"
            );
            return;
        }

        resultText.setText("🗣️ " + spokenText);
    }
}
```


## File: `app/src/main/AndroidManifest.xml`
Source blob SHA: `c042937139d4cc586f8078ade62951dd0a861a84`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.radin.assistant"
    android:versionCode="1"
    android:versionName="1.0">

    <uses-sdk android:minSdkVersion="23" android:targetSdkVersion="36" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <application
        android:allowBackup="false"
        android:label="@string/app_name"
        android:supportsRtl="true">

        <service
            android:name=".control.RadinAccessibilityService"
            android:exported="false"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/radin_accessibility_service" />
        </service>

        <activity
            android:name=".MainActivity"
            android:exported="true">

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

    </application>

</manifest>
```


## File: `app/src/main/res/xml/radin_accessibility_service.xml`
Source blob SHA: `33b2b02a44270c48363c03629f13b611517413d5`

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:canRetrieveWindowContent="true"
    android:canPerformGestures="true"
    android:description="@string/accessibility_service_description"
    android:notificationTimeout="100" />
```


## File: `app/build.gradle.kts`
Source blob SHA: `23f955dd63bd5a5a9af34ea42503c29a692ace30`

```kotlin
plugins {
    id("com.android.application") version "8.13.0"
}

android {
    namespace = "com.radin.assistant"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.radin.assistant"
        minSdk = 23
        targetSdk = 36
        versionCode = 2
        versionName = "1.1"
    }

    signingConfigs {
        create("radin") {
            val keystorePath = System.getenv("RADIN_KEYSTORE_PATH")
            if (!keystorePath.isNullOrBlank()) {
                storeFile = file(keystorePath)
                storePassword = System.getenv("RADIN_KEYSTORE_PASSWORD")
                keyAlias = System.getenv("RADIN_KEY_ALIAS") ?: "radin"
                keyPassword = System.getenv("RADIN_KEY_PASSWORD")
            }
        }
    }

    buildTypes {
        getByName("debug") {
            val signingPath = System.getenv("RADIN_KEYSTORE_PATH")
            if (!signingPath.isNullOrBlank()) {
                signingConfig = signingConfigs.getByName("radin")
            }
        }
    }
}
```


## File: `settings.gradle.kts`
Source blob SHA: `41fa0eb0976ef0fd013550e71a746958ba0442e2`

```kotlin
pluginManagement {
    repositories {
        google()
        maven { url = uri("https://maven.aliyun.com/repository/google") }
        mavenCentral()
        gradlePluginPortal()
    }

    resolutionStrategy {
        eachPlugin {
            if (requested.id.id == "com.android.application") {
                useModule("com.android.tools.build:gradle:${requested.version}")
            }
        }
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        maven { url = uri("https://maven.aliyun.com/repository/google") }
        mavenCentral()
    }
}

rootProject.name = "MohammadApp"
include(":app")
```


## File: `build.gradle.kts`
Source blob SHA: `4ab0787f35973f1d7330be0485c49b735eff11be`

```kotlin
/*
 * This file was generated by the Gradle 'init' task.
 *
 * This is a general purpose Gradle build.
 * Learn more about Gradle by exploring our Samples at https://docs.gradle.org/9.8.0/samples
 */
```


## File: `gradle/wrapper/gradle-wrapper.properties`
Source blob SHA: `e3aae55285f3feba455ca3c831cb930925571f2c`

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
networkTimeout=10000
retries=0
retryBackOffMs=500
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```


## File: `.github/workflows/diag-accessibility-detect.yml`
Source blob SHA: `592e0268b30f695aa8b65db2275f33e789440304`

```yaml
name: Accessibility Diagnostic Build

on:
  push:
    branches:
      - diag/accessibility-detect
    paths:
      - '.github/workflows/diag-accessibility-detect.yml'
      - 'app/**'
      - 'gradle/**'
      - 'gradlew'
      - 'build.gradle.kts'
      - 'settings.gradle.kts'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  assemble-debug:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout diagnostic branch
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Build unsigned diagnostic APK
        run: |
          chmod +x ./gradlew
          ./gradlew --no-daemon :app:assembleDebug --stacktrace

      - name: Upload diagnostic APK (unsigned)
        uses: actions/upload-artifact@v4
        with:
          name: radin-accessibility-diagnostic-unsigned
          path: app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: error
          retention-days: 3
```
