# android-webview-foldable-skia-fix
Fix and minimal reproduction for Chromium Skia GPU tile splitting &amp; ghost duplicate element glitches on Android foldables (Pixel Fold, Galaxy Fold) in WebView / Capacitor apps.

# Android WebView Foldable Skia Compositor Fix

> A solution and minimal reproduction for the duplicate/ghost element glitch caused by Chromium's Skia tile atlas compositor on Android foldables and multi-display devices when combining `backdrop-filter` with 3D hardware transforms.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Platforms: Android WebView](https://img.shields.io/badge/Platform-Android%20WebView%20%7C%20Capacitor%20%7C%20Ionic-green.svg)](#)

---

## 🔍 The Problem & Symptoms

On Android foldable devices (such as **Google Pixel Fold**, **Samsung Galaxy Z Fold**, and **OnePlus Open**) or multi-display emulators running Android API 34–36+, hybrid apps (Capacitor, Ionic, React Native WebView, Cordova, Tauri Mobile, or PWAs) frequently display **bizarre visual ghosting**:

- Modals, overlays, or dropdown menus appear **duplicated, torn, or overlaid on top of themselves**.
- The top half and bottom half of a popup overlap or render twin ghost copies of navigation links, buttons, or grid items.
- Tapping or scrolling causes the elements to flicker or duplicate further.

### 🤯 The Deceptive Part
When you connect **Chrome DevTools** to inspect the DOM:
- **There is only 1 element in the DOM.**
- There are no duplicate React components or re-rendering loops.
- Standard CSS "performance optimizations" like `transform: translate3d(0,0,0)`, `will-change: transform`, or `contain: layout paint` **make the tearing worse**.

---

## 🔬 Root Cause: Skia Tile Atlas Splitting

1. **Chromium's Skia Rendering Pipeline**: In modern Android WebView (Vulkan / GLES rasterization), hardware layers are allocated into Skia GPU texture tiles.
2. **Foldable Viewport Boundary**: On foldables and multi-display configurations, the compositor splits the surface across display partitions.
3. **The Crash of Two Features**:
   - `backdrop-filter: blur(...)` requires a multi-pass pixel readback of the background texture.
   - Hardware-promoted 3D transforms (`translate3d`, `isolation: isolate`, or `contain: paint`) force the layer into a hardware texture atlas.
   - On wide-aspect foldables and dual screens, Skia's texture tile atlas miscalculates the tile offset during the blur pass, causing the layer to be rasterized twice across the split boundary.

---

## 💡 The Solution

The fix is a targeted platform override:
1. **Preserve full glassmorphism** (`backdrop-filter`) for iOS (WebKit) and Desktop Chrome/Safari, where the compositor handles it flawlessly.
2. **On Android WebView**, cleanly bypass the broken Skia compositor pass by:
   - Setting `backdrop-filter: none !important;`
   - Removing 3D layer promotion (`transform: none !important;` or standard 2D `translateY`)
   - Removing `contain` and `isolation: isolate`
   - Using a high-opacity rgba background (e.g., `rgba(16, 20, 32, 0.95)` for dark mode or `rgba(255, 255, 255, 0.94)` for light mode) to deliver the same sleek look without triggering the Skia tile bug.

---

## 🛠️ CSS Implementation

```css
/* ==========================================================================
   Base styles (Desktop & iOS WebKit enjoy smooth backdrop-filter blur)
   ========================================================================== */
.modal-overlay {
  position: fixed;
  top: 10%;
  left: 50%;
  transform: translateX(-50%);
  width: min(520px, calc(100vw - 32px));
  background-color: rgba(20, 24, 38, 0.75);
  backdrop-filter: blur(25px);
  -webkit-backdrop-filter: blur(25px);
  border-radius: 12px;
  box-shadow: 0 12px 36px rgba(0, 0, 0, 0.6);
  z-index: 1000;
  animation: slideInDown 0.2s cubic-bezier(0.16, 1, 0.3, 1);
}

/* ==========================================================================
   Android Foldable & Multi-Display Skia Compositor Workaround
   ========================================================================== */
.is-android .modal-overlay,
.platform-android .modal-overlay {
  /* 1. Disable the backdrop-filter pass that causes tile splitting */
  backdrop-filter: none !important;
  -webkit-backdrop-filter: none !important;

  /* 2. Use a high-opacity solid color to maintain high-end aesthetic */
  background-color: rgba(16, 20, 32, 0.96) !important;

  /* 3. Strip 3D GPU layer promotion and compositor containment */
  transform: translateX(-50%) !important;
  -webkit-transform: translateX(-50%) !important;
  contain: none !important;
  isolation: auto !important;
  backface-visibility: visible !important;
  -webkit-backface-visibility: visible !important;

  /* 4. Use 2D opacity/translation animation instead of 3D transforms */
  animation: fadeIn 0.15s ease-out !important;
}

/* Light Theme support */
:root.theme-light.is-android .modal-overlay,
:root.theme-light.platform-android .modal-overlay {
  background-color: rgba(255, 255, 255, 0.95) !important;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

---

## 📱 Platform Detection Snippet (Capacitor / Web)

```javascript
// Automatically add platform class to <html> on startup
import { Capacitor } from '@capacitor/core';

if (Capacitor.getPlatform() === 'android') {
  document.documentElement.classList.add('is-android');
}
```

---

## 🧪 Testing on Emulators

To reproduce and verify the fix:
1. Open **Android Studio > Virtual Device Manager**.
2. Launch a **Pixel Fold** or **Pixel 9 Pro Fold** running Android 14+ (API 34+).
3. Test with the device **unfolded** (dual-pane aspect ratio).
4. Trigger your overlay menu or modal.

---

## 📄 License
MIT © Ian B
