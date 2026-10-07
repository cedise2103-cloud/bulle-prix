name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main]
permissions:
  contents: write
env:
  JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
  LC_ALL: C.UTF-8
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'
      - name: Generate project
        run: |
          mkdir -p VeepeeBubble
          cat > VeepeeBubble/README.txt <<'FILE_EOF'
          Bulle Prix - prototype (lecture de l'ecran via accessibilite)
          FILE_EOF
          mkdir -p VeepeeBubble/app
          cat > VeepeeBubble/app/build.gradle.kts <<'FILE_EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "fr.ced.bubble"
              compileSdk = 34
              defaultConfig {
                  applicationId = "fr.ced.bubble"
                  minSdk = 26
                  targetSdk = 34
                  versionCode = 1
                  versionName = "0.1"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          FILE_EOF
          mkdir -p VeepeeBubble/app/src/main
          cat > VeepeeBubble/app/src/main/AndroidManifest.xml <<'FILE_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <application
                  android:label="@string/app_name"
                  android:theme="@android:style/Theme.Material.Light">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <service
                      android:name=".PriceAccessibilityService"
                      android:exported="true"
                      android:label="@string/app_name"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          FILE_EOF
          mkdir -p VeepeeBubble/app/src/main/java/fr/ced/bubble
          cat > VeepeeBubble/app/src/main/java/fr/ced/bubble/MainActivity.kt <<'FILE_EOF'
          package fr.ced.bubble

          import android.app.Activity
          import android.content.Intent
          import android.os.Bundle
          import android.provider.Settings
          import android.widget.Button
          import android.widget.EditText
          import android.widget.LinearLayout
          import android.widget.TextView
          import android.widget.Toast

          class MainActivity : Activity() {
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val prefs = getSharedPreferences("cfg", MODE_PRIVATE)
                  val pad = (20 * resources.displayMetrics.density).toInt()

                  val layout = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      setPadding(pad, pad, pad, pad)
