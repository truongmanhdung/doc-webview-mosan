# Mosan End-User App — React Native WebView Integration Details

**To:** Partner team  
**From:** Mosan / Telemor mobile team  
**Re:** E-Commerce WebView reload issue — information requested to narrow root cause  
**Screen:** `ECommerceWeb` (E-Mercado / shop flow)

---

## Context

We are investigating a **continuous reload** issue when opening your website inside our React Native app on certain devices.

To determine whether the issue is on the **React Native / WebView wrapper side** or the **website side**, please find below the exact library version and WebView configuration we use.

**Additional observation (our side):** We tested an **APK built with Kotlin Multiplatform** on the same affected device. The website **works correctly after opening it a second time**. Our React Native app uses `incognito={true}` on the WebView (no persistent session), which may behave differently from your KMP WebView — we are happy to align or test changes together.

---

## 1. react-native-webview version

| Item | Value |
|------|--------|
| Package name | `react-native-webview` |
| Declared in `package.json` | `^11.17.2` |
| **Resolved / installed version** (`yarn.lock`) | **`11.26.1`** |
| React Native | `0.68.5` |
| Platform | iOS and Android (Mosan end-user app) |

---

## 2. WebView configuration — how we load your website

### 2.1 Source file

`apps/end-user/src/screens/ECommerce/ECommerceWeb/index.tsx`

### 2.2 How the URL is obtained

1. Our app calls the Mosan backend API (`ECommerceServices.checkInfoAndGetUrl`) to receive the shop URL (`shopInfo.url`).
2. Before showing the WebView, we run a plain `fetch(shopInfo.url)` to check the HTTP response (if status is not 200, we show an error modal and do not mount the WebView).
3. The WebView is rendered only when:
   - `shopInfo.url` is available,
   - the session is not timed out, and
   - the pre-check did not fail.

### 2.3 WebView snippet (actual code)

```tsx
import { WebView } from 'react-native-webview';

const source = { uri: `${shopInfo?.url}` }; // URL from Mosan API

<WebView
  ref={refWebview}
  source={source}
  originWhitelist={['*']}
  style={{ width: '100%', flex: 1, borderRadius: 0 }}
  javaScriptEnabled={true}
  startInLoadingState={false}
  cacheEnabled={true}
  incognito={true}
  domStorageEnabled={true}
  mixedContentMode="compatibility"
  bounces={false}
  scrollEnabled={true}
  androidHardwareAccelerationDisabled={false}
  renderToHardwareTextureAndroid
  javaScriptCanOpenWindowsAutomatically
  onLoadStart={showLoading}
  onLoadEnd={hideLoading}
  renderLoading={renderLoading}
  onNavigationStateChange={onNavigationStateChange}
  onMessage={(event) => {
    console.log('Received message from WebView:', event.nativeEvent.data);
  }}
  onError={(syntheticEvent) => {
    console.warn('WebView error: ', syntheticEvent.nativeEvent);
  }}
/>
```

### 2.4 Important integration details

| Topic | Our implementation |
|--------|---------------------|
| **Custom HTTP headers** | **None** — we do not pass `Authorization`, cookies, or a custom `userAgent` in `source`. |
| **Cookies / session** | `domStorageEnabled={true}`, but **`incognito={true}`** (private browsing — limited persistent cache/session). |
| **JavaScript** | Enabled (`javaScriptEnabled={true}`). |
| **Mixed content** | `mixedContentMode="compatibility"`. |
| **Return to native app** | `onNavigationStateChange`: if URL contains the string **`BackToApp`**, we navigate to the native confirmation screen or reset navigation. |
| **Screen focus** | On navigation `focus`, we call `refWebview.current?.goBack()` (intended to reset WebView history when re-entering the screen). |
| **postMessage** | We log messages from the WebView but do not inject JavaScript from native in this screen. |

### 2.5 Related screen (same WebView pattern)

We use a similar WebView setup on **Lotto** (`apps/end-user/src/screens/Lotto/Lotto/index.tsx`) with the same core props (`incognito`, `cacheEnabled`, `domStorageEnabled`, etc.).

---

## 3. Contact

For a test URL, device model, or log correlation (timestamps / account), please coordinate with our mobile team.

**Package reference:** `mosan-mobileapp-main` (monorepo root `package.json` → `react-native-webview`)
