### 1. Prefer _Pressable_ over _TouchableOpacity_

**TouchableOpacity** is outdated. **Pressable** offers more versatility, allowing you to handle more events like `onPressIn`, `onPressOut`, and `onLongPress` (1:43).

- **Detailed Guide:**
    - Replace all instances of `TouchableOpacity` with `Pressable`.
    - For better user experience, consider using **Presto**, a library built on top of `Pressable` that easily adds haptic feedback and animations to your buttons and images without extra configuration.
    - Configure global haptics at the entry point of your application.

### 2. Use Platform File Extensions, Not Runtime Checks

Avoid using the `Platform` API for inline checks (e.g., `Platform.OS === 'ios'`). Instead, use file extensions to keep native code separate (4:02).

- **Detailed Guide:**
    - For platform-specific components or screens, create separate files. For example, use `ComponentName.ios.tsx` and `ComponentName.android.tsx`.
    - React Native will automatically import the correct file based on the platform you are compiling for.
    - This keeps your main codebase clean and makes platform-specific optimizations easier to manage.

### 3. Use Sheets Instead of Modals

While the `Modal` component works, **presentation sheets** (specifically on iOS) offer a better native experience with modern UI features like blurred backgrounds.

- **Detailed Guide:**
    - Use a form sheet to present context on top of your screen.
    - Configure `detents` to control how high the sheet expands (e.g., 45% of the screen).
    - On Android, use a modal with a `slideFromBottom` animation to emulate this behavior.

### 4. Prefer _FlatList_ over _ScrollView_ for Large Lists

**ScrollView** renders all items at once, which is bad for performance. **FlatList** (or **FlashList**) only renders items currently visible on the screen (8:53).

- **Detailed Guide:**
    - Use `FlatList` for any list that could contain a significant number of items.
    - Utilize `FlatList` properties like `ListEmptyComponent` for better UX when there is no data.
    - Instead of `SafeAreaView`, use `contentInsetAdjustmentBehavior="automatic"` on your list component to handle headers and safe areas automatically.

### 5. Keep the _/app_ Directory Focused on Routing

If you are using Expo Router or React Navigation, your route files should only handle navigation logic, not complex business logic.

- **Detailed Guide:**
    - Your `/app` folder (or route definition files) should be "lean." Keep them focused on layout and navigation configuration.
    - Extract screen content and business logic into separate component files within a `/components` or `/screens` directory.
    - This makes refactoring and reusing components much easier.

### **6. Use `ignoreSafeArea` Property** 

If UI components are breaking or behaving strangely near the **keyboard** or screen edges, use this property to control how the _SwiftUI hosting view_ reacts.

- **Guide:** On your `Host` component, add `ignoreSafeArea={['keyboard']}` to prevent the view from jumping when the keyboard opens, or `ignoreSafeArea={['all']}` to allow elements to overlap top/bottom safe areas.

### **7. Implement `matchContents` Property** 

Instead of manually calculating `height` or `width` for _SwiftUI_ views, which often leads to broken layouts or unresponsive touch areas, use this property.

- **Guide:** Remove manual styling (`width`, `height`, `flex`) from the `Host` component and simply add `matchContents={true}`. This tells the view to automatically size itself based on the native _SwiftUI_ content inside.

### **8. Utilize `ReactNativeHostView`** 

When working inside a _SwiftUI_ or _Jetpack Compose_ context, standard _React Native_ views (``) might not render properly. Use the special host view to bridge them.

- **Guide:** Import `ReactNativeHostView` from `expo-ui`. Wrap your standard _React Native_ components within this view to render them successfully inside native _SwiftUI_ containers.


### **9. Centralize Your API Logic** 

Avoid duplicating data fetching or state management logic across different platform implementations.

- **Guide:** Create a **context provider** or a dedicated service file to handle fetching and logic. Import this shared logic into your specific iOS and Android components to maintain a single source of truth.

### **10. Utilize Platform Colors** 

Make your app feel truly native by automatically adapting to user system settings, including **light/dark mode** and **material colors** on Android.

- **Guide:** Use the `color` utility from _Expo Router_ to access native system colors (e.g., `color.ios.systemBackground` or `color.android.dynamic.surface`). On Android, this will pull color palettes directly from the user's wallpaper.

### **11. Native Tabs with System Behaviors** 

- **Summary:** Utilize the system tab bar on both iOS and Android to automatically support system-level behaviors like "liquid glass" and tab bar minimization on scroll.
- **Guide:**
    - In your **layout file**, configure your `Tabs` component.
    - Use the `minimizeBehavior` prop to make the tab bar hide automatically when users scroll down.
    - Use the `rollSearch` prop for special tabs (like a camera tab) to visually separate them from the rest of the tab bar.
    - Manually set the `selected` prop to define the active state icon (e.g., `camera-fill`) versus the inactive state (`magnifyingglass`).
### **12. Stack Toolbar API** 
- **Summary:** Use native primitives to place buttons, menus, and spacers in the header or footer without relying on hacks.
- **Guide:**
    - Implement the `StackToolbar` component within your screen options.
    - Use the `placement` prop to position items on the `left`, `right`, or `bottom` of the screen.
    - Set the `tintColor` to match your app's theme to get the native glass effect on iOS 26.**3. Bottom Accessory (Mini Player)** 
- **Summary:** Attach interactive components (like a music mini-player) that sit above the tab bar and animate smoothly.
- **Guide:**
    - Implement `NativeTabs.BottomAccessory` within your root tab layout.
    - Place your custom view (e.g., a `Pressable` with album art and controls) inside this component.
    - This accessory will automatically minimize along with the tab bar.
### **13. Zoom Transitions** 
- **Summary:** Create continuous, interactive animations when navigating between images or thumbnails and detail screens.
- **Guide:**
    - Wrap your navigation `Link` or `Pressable` with the `appleZoom` behavior.
    - On the destination screen, use `Link.AppleZoomTarget` on the component (e.g., the large image) that should be the focus of the transition.
    - This allows users to interrupt and drag the animation back.
### **14. Form Sheets with Flex-1 Fix** 
- **Summary:** Fix layout issues when pinning content to the bottom of native form sheets using standard CSS `flex: 1`.
- **Guide:**
    - When presenting a screen as a `formSheet`, you can now use a view with `flex: 1` to containerize your content.
    - Pin action buttons or informational text to the bottom of the screen using standard flexbox alignment without worrying about layout jumps.