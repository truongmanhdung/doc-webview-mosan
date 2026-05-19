# Mosan End-User App — React Native WebView Integration Details

**To:** Partner team  
**From:** Mosan / Telemor mobile team  
**Re:** E-Commerce WebView reload issue — information requested to narrow root cause  
**Screen:** `ECommerceWeb` (E-Mercado / shop flow)  
**Source file:** `apps/end-user/src/screens/ECommerce/ECommerceWeb/index.tsx`

---

## Context

We are investigating a **continuous reload** issue when opening your website inside our React Native app on certain devices (e.g. Vivo X70 Pro, Android 14). The issue often appears from the **second visit** onward.

To determine whether the issue is on the **React Native / WebView wrapper side** or the **website side**, please find below the exact library version, WebView configuration, and **code that affects the WebView** only.

**Additional observation (our side):** We tested an **APK built with Kotlin Multiplatform** on the same affected device. The website **works correctly after opening it a second time**. We have adjusted our React Native integration (see section 2.4) to reduce Android-specific reload behavior.

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

1. **Self-buy flow** (`fetchSelfBuyUrl: true`): the app calls Mosan backend API `ECommerceServices.checkInfoAndGetUrl` via `useECommerce().onCheckInfo({ selfBuy: true })` and stores the result in `fetchedInfo.url`.
2. **Buy-for-others / skip flow**: the URL is passed in navigation params as `route.params.info.url`.
3. `shopInfo.url` is either `paramInfo.url` or `fetchedInfo.url`.
4. The WebView is rendered only when:
   - `shopInfo.url` is available,
   - `sessionReducer.isTimeOut` is false,
   - `errorWebView` is false.
5. We **do not** run a separate `fetch(url)` before mounting the WebView. Load errors are handled via WebView `onError`.

### 2.3 WebView code

See **section 3** for all code that affects the WebView: API to get URL (3.1), mount guard (3.2), `BackToApp` callback (3.3), loading (3.4), `<WebView />` props (3.5).

### 2.4 Important integration details

| Topic | Our implementation |
|--------|---------------------|
| **Custom HTTP headers** | **None** — we do not pass `Authorization`, cookies, or a custom `userAgent` in `source`. |
| **Cookies / session** | `domStorageEnabled={true}`. **`incognito` is enabled on iOS only** (`incognito={Platform.OS === 'ios'}`). On Android, cache/session can persist between visits. |
| **JavaScript** | Enabled (`javaScriptEnabled={true}`). |
| **Mixed content** | `mixedContentMode="compatibility"`. |
| **WebView instance** | `key={shopInfo.url}` — remount when URL changes. |
| **Return to native app** | `onNavigationStateChange`: if URL contains **`BackToApp`**, navigate to `ECommerceConfirmDetail` or reset navigation. Handled **once per screen visit** via `backToAppHandled` ref. |
| **Screen focus** | We **do not** call `webview.goBack()` on navigation focus (removed to avoid Android reload loops on re-entry). |
| **Pre-fetch URL** | **Removed** — single load via WebView only. |
| **postMessage** | We log messages from the WebView but do not inject JavaScript from native on this screen. |
| **Loading overlay** | Global `showLoading` / `hideLoading` on `onLoadStart` / `onLoadEnd`. |

### 2.5 Related screen

We use a similar WebView setup on **Lotto** (`apps/end-user/src/screens/Lotto/Lotto/index.tsx`). E-Commerce uses the configuration above.

---

## 3. Code that affects the WebView

Excerpt from `ECommerceWeb/index.tsx` and related helpers — only logic tied to obtaining the URL, mounting the WebView, and handling return from the website.

### 3.1 API call to get shop URL

**Endpoint:** `POST {api_base}/emercado/get-url`  
**Service:** `libs/services/src/ECommerceService.tsx` → `ECommerceServices.checkInfoAndGetUrl(body)`

**Request body (`ILottoCheckInfo`):**

| Field | Description |
|--------|-------------|
| `phoneNumber` | User account number (self-buy) or recipient phone (buy for others) |
| `selfBuy` | `true` = buy for self, `false` = buy for others |
| `signature` | SHA256 hash signed with app `signatureKey` |

**Response used for WebView:** `response.data.info.url` and `response.data.transId`

**Helper** (`libs/helpers/src/features/eCommerce.tsx`):

```tsx
const onCheckInfo = async (param: ILottoCheckInfo) => {
  showLoading();
  const { phoneNumber, selfBuy } = param;
  let signature;
  if (selfBuy) {
    signature = await EncryptHelper.encryptSha256(
      `${authenticationReducer?.userInfo?.accountNumber}${selfBuy}`,
      authenticationReducer.signatureKey ?? ''
    );
  } else {
    signature = await EncryptHelper.encryptSha256(
      `${phoneNumber}${selfBuy}`,
      authenticationReducer.signatureKey ?? ''
    );
  }
  const params = {
    phoneNumber: selfBuy
      ? authenticationReducer?.userInfo?.accountNumber
      : phoneNumber,
    selfBuy,
    signature,
  };
  const response = await ECommerceServices.checkInfoAndGetUrl(params);
  hideLoading();
  if (response?.succeeded && !response.failed) {
    return { success: true, data: response.data };
  }
  return { success: false, message: response?.data?.message, data: response?.data };
};
```

**Flow A — Self-buy inside `ECommerceWeb`** (`fetchSelfBuyUrl: true`, no URL in params):

```tsx
const { onCheckInfo } = useECommerce();
const fetchSelfBuyUrl = route.params?.fetchSelfBuyUrl === true;
const [fetchedInfo, setFetchedInfo] = useState<ILottoInfo | undefined>();

useEffect(() => {
  if (!fetchSelfBuyUrl || paramInfo?.url) {
    return;
  }
  let cancelled = false;
  const load = async () => {
    const response = await onCheckInfo({ selfBuy: true });
    if (cancelled) return;
    if (response?.success && response?.data && !sessionReducer?.isTimeOut) {
      setFetchedInfo({
        selfBuy: true,
        url: response?.data?.info?.url,
        transId: response?.data?.transId,
      });
    } else {
      // show fetch error modal
    }
  };
  void load();
  return () => { cancelled = true; };
}, [fetchSelfBuyUrl, paramInfo?.url]);
```

**Flow B — URL already from previous screen** (`ECommerceEnterInformation` calls API, then navigates with `info.url`):

```tsx
const response = await onCheckInfo({
  phoneNumber: phoneNumberAccountString,
  selfBuy: selfBuy ?? false,
});
if (response?.success && response?.data) {
  navigation.navigate('ECommerceWeb', {
    info: {
      selfBuy: selfBuy ?? false,
      url: response?.data?.info?.url,
      transId: response?.data?.transId,
    },
  });
}
```

After the URL is set, `shopInfo.url` is passed to the WebView as `source={{ uri: shopInfo.url }}`.

---

### 3.2 URL source and mount guard

```tsx
const refWebview = useRef<WebView>(null);
const paramInfo = route.params?.info;
const [errorWebView, setErrorWebView] = useState(() => !paramInfo?.url);
const [fetchedInfo, setFetchedInfo] = useState<ILottoInfo | undefined>();

const shopInfo = paramInfo?.url ? paramInfo : fetchedInfo;
const sessionReducer = useAppSelector((state) => state.SessionReducer);

const source = { uri: `${shopInfo?.url}` };

// WebView mounts only when all conditions are true:
{!sessionReducer?.isTimeOut && !errorWebView && shopInfo?.url && (
  <WebView /* ... */ />
)}
```

When `shopInfo.url` becomes available, we allow the WebView to show:

```tsx
useEffect(() => {
  if (!shopInfo?.url) {
    return;
  }
  setErrorWebView(false);
}, [shopInfo?.url]);
```

### 3.3 Callback when returning from website (`BackToApp`)

Your website should redirect or navigate to a URL that **contains the string `BackToApp`**. We detect it in `onNavigationStateChange` (fires on every in-WebView navigation).

**Contract:**

| Item | Value |
|------|--------|
| Trigger | `webViewState.url` includes `'BackToApp'` |
| Frequency | Handled **once** per screen visit (`backToAppHandled` ref) |
| Data passed to native | Full callback URL in `info.url` for confirmation screen |

**Full handler:**

```tsx
const backToAppHandled = useRef(false);
const userInfo = useAppSelector((state) => state.AuthenticationReducer.userInfo);
const selfBuy = shopInfo?.selfBuy ?? selfBuyParam;

const onNavigationStateChange = (webViewState: { url?: string }) => {
  console.info('URL-------', webViewState);
  if (
    !backToAppHandled.current &&
    `${webViewState?.url}`.includes('BackToApp')
  ) {
    backToAppHandled.current = true;

    if (userInfo) {
      // Logged-in user → confirm payment / order on native screen
      navigation.navigate('ECommerceConfirmDetail', {
        info: {
          ...shopInfo,
          url: `${webViewState?.url}`,
        },
      });
    } else {
      // Guest / skip-login flow
      hideLoading();
      dispatch(AuthenticationActions.setUserSkipNow.request('1'));

      if (selfBuy) {
        navigation.reset({
          routes: [
            { name: 'TabRoute' },
            {
              name: 'ECommerceWeb',
              params: { fetchSelfBuyUrl: true, selfBuy: true },
            },
          ],
        });
      } else {
        navigation.reset({
          routes: [
            { name: 'TabRoute' },
            {
              name: 'ECommerceEnterInformation',
              params: {
                info: { selfBuy: false },
                phoneNumber: phoneNumberSkipNow,
              },
            },
          ],
        });
      }
    }
  }
};

// Wired on WebView:
// onNavigationStateChange={onNavigationStateChange}
```

**Note for partner:** Please ensure only the **final** checkout/return URL contains `BackToApp`. Intermediate redirects or query parameters on other pages should not include that string, or our app may leave the WebView early.

---

### 3.4 Loading indicator (WebView callbacks)

```tsx
const renderLoading = () => (
  <View style={styles.loading}>
    <ActivityIndicator size="large" color={theme.colors?.primary} />
  </View>
);

// Used on WebView:
// onLoadStart={showLoading}
// onLoadEnd={hideLoading}
// renderLoading={renderLoading}
```

### 3.5 `<WebView />` component (full)

```tsx
<WebView
  key={shopInfo.url}
  ref={refWebview}
  source={source}
  originWhitelist={['*']}
  style={{ width: '100%', flex: 1, borderRadius: 0 }}
  javaScriptEnabled={true}
  startInLoadingState={false}
  cacheEnabled={true}
  incognito={Platform.OS === 'ios'}
  domStorageEnabled={true}
  mixedContentMode="compatibility"
  bounces={false}
  scrollEnabled={true}
  androidHardwareAccelerationDisabled={false}
  javaScriptCanOpenWindowsAutomatically
  onLoadStart={showLoading}
  onLoadEnd={hideLoading}
  renderLoading={renderLoading}
  onNavigationStateChange={onNavigationStateChange}
  onMessage={(event) => {
    console.log('Received message from WebView:', event.nativeEvent.data);
  }}
  onError={(syntheticEvent) => {
    const { nativeEvent } = syntheticEvent;
    console.warn('WebView error: ', nativeEvent);
    setErrorWebView(true);
    // show error modal
  }}
/>
```

---

## 4. Contact

For a test URL, device model, or log correlation (timestamps / account), please coordinate with our mobile team.

**Package reference:** `mosan-mobileapp-main` (monorepo root `package.json` → `react-native-webview`)

---

## 5. Software changes — continuous WebView reload mitigation

**Context:** On some Android devices (e.g. Vivo X70 Pro, Android 14), the shop WebView could reload repeatedly, often starting from the **second visit** to the screen.

**Source file:** `apps/end-user/src/screens/ECommerce/ECommerceWeb/index.tsx`

| # | Change | Rationale |
|---|--------|-----------|
| 1 | **Removed** `navigation.addListener('focus', () => refWebview.current?.goBack())` | On Android, calling `goBack()` on every screen **focus** often **reloads** the page. After the first visit the WebView has history, so each refocus can trigger a reload loop (see [react-native-webview #3933](https://github.com/react-native-webview/react-native-webview/issues/3933)). |
| 2 | **Removed** pre-request `fetch(shopInfo.url)` before mounting the WebView | Avoids **hitting the URL twice** (standalone `fetch` + WebView load) and avoids toggling `errorWebView` in a way that **unmounts/remounts** the WebView repeatedly on some devices. |
| 3 | **`errorWebView`** initial state: `useState(() => !paramInfo?.url)` | If the URL is already in `route.params`, do not block the first paint behind a wrong gate → fewer unnecessary remounts. |
| 4 | **`incognito={Platform.OS === 'ios'}`** (incognito off on Android) | Android incognito behavior is inconsistent; disabling it on Android keeps **cookies/session** closer to a normal browser (similar to KMP where a second open worked). |
| 5 | **Removed** `renderToHardwareTextureAndroid` | Some OEM devices (Vivo, etc.) show WebView glitches or reloads when this is enabled. |
| 6 | Added **`backToAppHandled`** (`useRef`) in `onNavigationStateChange` | `BackToApp` may appear across multiple redirects → handle **once** per screen visit to avoid tight `navigate` / `reset` loops. |
| 7 | Added **`key={shopInfo.url}`** on `<WebView />` | Remount the WebView **only when the shop URL changes**, not on unrelated re-renders. |
| 8 | Load failures handled via WebView **`onError`** (error modal) | Replaces relying only on pre-`fetch` status; aligns with row 2. |

**Implementation note:** The same points are summarized in the block comment at the top of `ECommerceWeb/index.tsx` for quick internal reference.
