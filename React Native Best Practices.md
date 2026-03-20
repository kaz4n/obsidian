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