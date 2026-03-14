# Formula — App Store Submission Guide

## Prerequisites

### Apple App Store (iOS)
- **Apple Developer Account** ($99/year): https://developer.apple.com/programs/
- **Xcode** (installed)
- **App Store Connect** access: https://appstoreconnect.apple.com

### Google Play Store (Android)
- **Google Play Developer Account** ($25 one-time): https://play.google.com/console/signup
- **Android Studio**: https://developer.android.com/studio
- **Java JDK 17+**

---

## iOS — App Store Submission

### Step 1: Set Up Signing

1. Open the project in Xcode:
   ```bash
   npx cap open ios
   ```

2. In Xcode, select the **App** target → **Signing & Capabilities**
3. Check **Automatically manage signing**
4. Select your **Team** (your Apple Developer account)
5. The bundle identifier should be `app.prv8.formula`

### Step 2: Create App in App Store Connect

1. Go to https://appstoreconnect.apple.com
2. Click **My Apps** → **+** → **New App**
3. Fill in:
   - **Platform**: iOS
   - **Name**: Formula
   - **Primary Language**: English (U.S.)
   - **Bundle ID**: app.prv8.formula
   - **SKU**: formula-volleyball-analytics

### Step 3: Build Archive

1. In Xcode, select **Product** → **Archive**
   - Make sure the scheme is set to **App** and destination is **Any iOS Device**
2. When the archive completes, the **Organizer** window opens
3. Click **Distribute App**
4. Select **App Store Connect** → **Upload**
5. Follow the prompts to upload

### Step 4: Submit for Review

1. In App Store Connect, go to your app
2. Fill in the required metadata:
   - **Description**: Volleyball analytics platform featuring PIR (Player Impact Rating) and Winning Formula — proprietary metrics that provide coaches with actionable statistical insights.
   - **Keywords**: volleyball, analytics, stats, coaching, PIR, sports, statistics, player rating
   - **Support URL**: https://formula.prv8.app
   - **Category**: Sports
   - **Age Rating**: 4+
   - **Price**: Free (or your chosen price)
3. Upload **screenshots** (required sizes):
   - iPhone 6.9" (1320 x 2868)
   - iPhone 6.7" (1290 x 2796)
   - iPad Pro 13" (2064 x 2752)
4. Add a **Privacy Policy URL** (required)
5. Click **Submit for Review**

### Taking Screenshots
Use the iOS Simulator to capture screenshots:
```bash
# Run in simulator
npx cap run ios --target "iPhone 16 Pro Max"

# Take screenshots using Cmd+S in the simulator
```

---

## Android — Google Play Store Submission

### Step 1: Install Android Studio & SDK

1. Download Android Studio: https://developer.android.com/studio
2. Install and open it
3. Go to **SDK Manager** → install **Android SDK 34**
4. Set `ANDROID_HOME` environment variable:
   ```bash
   export ANDROID_HOME=$HOME/Library/Android/sdk
   export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
   ```

### Step 2: Open Project

```bash
npx cap open android
```

### Step 3: Generate Signed APK/AAB

1. In Android Studio: **Build** → **Generate Signed Bundle / APK**
2. Select **Android App Bundle** (required for Play Store)
3. Create a new keystore or use existing:
   - **Keystore path**: Choose a secure location
   - **Keystore password**: Create a strong password
   - **Key alias**: formula
   - **Key password**: Create a strong password
   - **Validity**: 25 years
   - **Certificate info**: Fill in your details
4. Select **Release** build variant
5. Click **Finish**

**IMPORTANT**: Back up your keystore file and passwords. You need the same keystore for all future updates.

### Step 4: Create App in Google Play Console

1. Go to https://play.google.com/console
2. Click **Create app**
3. Fill in:
   - **App name**: Formula
   - **Default language**: English (United States)
   - **App or game**: App
   - **Free or paid**: Free (or your choice)

### Step 5: Store Listing

Fill in the required information:
- **Short description** (80 chars max): Volleyball analytics with PIR ratings and Winning Formula metrics.
- **Full description**: Formula is a volleyball analytics platform featuring PIR (Player Impact Rating) and Winning Formula — proprietary metrics that provide coaches with actionable statistical insights. Track player performance, analyze team statistics, and discover the winning formula for your team.
- **Screenshots**: At least 2 phone screenshots (16:9 or 9:16 ratio)
- **Feature graphic**: 1024 x 500 PNG
- **App icon**: 512 x 512 (already generated)
- **Category**: Sports
- **Contact email**: Your email
- **Privacy Policy URL**: Required

### Step 6: Upload & Submit

1. Go to **Release** → **Production**
2. Click **Create new release**
3. Upload the `.aab` file from Step 3
4. Add release notes
5. Click **Review release** → **Start rollout to production**

---

## Quick Build Commands

```bash
# Build web assets and sync to native projects
npm run cap:build

# Open iOS project in Xcode
npm run cap:open:ios

# Open Android project in Android Studio
npm run cap:open:android

# Sync after making web changes
npm run cap:sync
```

---

## Privacy Policy Requirement

Both stores require a privacy policy. Your app collects:
- Team/player statistics data (stored in Firebase)

Create a privacy policy page at your domain (e.g., https://formula.prv8.app/privacy) covering:
- What data is collected
- How data is stored (Firebase)
- Data retention policy
- Contact information

---

## Version Updates

When releasing updates:

1. Update version in `capacitor.config.ts`
2. Update version in `package.json`
3. For iOS: Update in Xcode (General → Identity → Version/Build)
4. For Android: Update `versionCode` (increment by 1) and `versionName` in `android/app/build.gradle`
5. Run `npm run cap:build`
6. Archive and upload through Xcode / Android Studio
