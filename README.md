# Deployment-Workflows
```yml

name: Deploy

on:
  pull_request:
    branches: [ main]
    types: [closed]

jobs:

    build:
     if: github.event.pull_request.merged == true
     name: 🚀 Deploy
     runs-on: ubuntu-latest
     env:
       SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
     steps:

      # Check out Code
        - name: Checkout code
          uses: actions/checkout@v2
          with:
            ref: ${{ github.event.inputs.branch }}
      
      # Set up JDK
        - name: Set up JDK 11
          uses: actions/setup-java@v1
          with:
             java-version: 11

      # Set Up Android SDK (Optional)
        - name: Setup Android SDK
          uses: android-actions/setup-android@v2

      # Gradle Caching (Optional)
        - name: Gradle Cache
          uses: actions/cache@v2
          with:
            path: |
               ~/.gradle/caches
               ~/.gradle/wrapper
               key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
               restore-keys: |
               ${{ runner.os }}-gradle-

      # Grants execute permission
        - name: Grant rights to gradlew
          run: chmod +x ./gradlew

      # override the version code and version name
        - name: Bump version code
          uses: chkfung/android-version-actions@v1.1
          with:
            gradlePath: app/build.gradle
            versionCode: ${{github.run_number}}
      
      # Build the app with gradle
        - name: Build with gradle
          id: build
          run: ./gradlew build --stacktrace

      # Run Unit text
        - name: Unit Test
          id: test
          run: ./gradlew test


      # Build App Bundle
        - name: Build Release AAB
          id: buildRelease
          run: ./gradlew bundleRelease --stacktrace

      # Build Apk (optional)
        - name: Generate app APK.
          id: buildApk
          run: ./gradlew assembleRelease --stacktrace  

      # Upload App Bundle (optional)
        - name: Upload AAB
          id: uploadArtifact
          uses: actions/upload-artifact@v1
          with:
            name: app
            path: app/build/outputs/bundle/release/app-release.aab  

      # Sign App Bundle
        - name: Sign AAB
          id: sign
          uses: r0adkll/sign-android-release@v1
          with:
             releaseDirectory: app/build/outputs/bundle/release
             signingKeyBase64: ${{ secrets.SIGNING_KEY }}
             alias: ${{ secrets.ALIAS }}
             keyStorePassword: ${{ secrets.KEY_STORE_PASSWORD }}
             keyPassword: ${{ secrets.KEY_PASSWORD }}

      # Service Account Json
        - name: Create service_account.json
          id: createServiceAccount
          run: echo '${{ secrets.SERVICE_ACCOUNT_JSON }}' > service_account.json

      # Uploads the .aad to play console
        - name: Deploy to Play Store
          id: deploy
          uses: r0adkll/upload-google-play@v1.0.15
          with:
            serviceAccountJson: service_account.json
            packageName: com.appdeveloper.AppName
            releaseFile: app/build/outputs/bundle/release/app-release.aab
            track: internal

      # Slack Notificatin
        - name: Slack Notification
          uses: act10ns/slack@v1
          with:
            status: ${{ job.status }}
            steps: ${{ toJson(steps) }}
          if: always()

  
```
