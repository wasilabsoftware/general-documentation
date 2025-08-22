v# Styling Documentation - TicTacHub

## Overview

TicTacHub uses **React Native Unistyles v3** as its primary styling system. Unistyles is a powerful, performant styling library that provides runtime theming, responsive breakpoints, and TypeScript-safe styles without the performance overhead of traditional hooks-based theming solutions.

## Version Information

- **React Native Unistyles**: v3.0.5
- **React Native**: 0.79.5
- **Expo**: ~53.0.20

## Setup and Configuration

### 1. Babel Configuration

The Babel plugin is essential for Unistyles to work properly. It processes your styles at build time for optimal performance.

**File: `babel.config.js`**

```javascript
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      ['react-native-unistyles/plugin', {
        // Root folder where all files will be processed by the Babel plugin
        root: 'src'
      }]
    ]
  };
};
```

**Key Points:**
- The `root` option specifies which folder contains your application code
- All files under this folder will be processed by the Babel plugin
- The plugin enables compile-time optimizations and proper style extraction

### 2. Initialization

Unistyles requires one-time initialization at app startup. This is done through a dedicated initialization file that's imported at the root of the application.

**File: `unistyles-init.ts`**

```typescript
// This file ensures Unistyles is configured before any components are imported
import '@/src/styles/unistyles';
```

**File: `app/_layout.tsx`**

```typescript
import '../unistyles-init';  // Must be the first import!
import { Stack } from "expo-router";
// ... rest of imports
```

**Important:** The initialization import MUST be the first import in your root layout file to ensure Unistyles is configured before any components are loaded.

### 3. Theme Configuration

**File: `src/styles/unistyles.ts`**

This file contains the complete Unistyles configuration including themes, breakpoints, and settings.

```typescript
import { StyleSheet } from 'react-native-unistyles'
import { colors, spacing, typography, borderRadius } from './colors'

// Configure themes
const darkTheme = {
  colors: {
    background: colors.primary.indigo900,
    primary: colors.primary.purple600,
    // ... all theme tokens
  },
  spacing: { /* spacing system */ },
  typography: { /* typography system */ },
  borderRadius: { /* border radius system */ },
  components: { /* component-specific tokens */ }
}

const lightTheme = {
  ...darkTheme,
  colors: {
    ...darkTheme.colors,
    // Light theme overrides
  }
}

// Configure breakpoints
const breakpoints = {
  xs: 0,      // Mobile phones
  sm: 576,    // Large phones
  md: 768,    // Tablets
  lg: 992,    // Desktop
  xl: 1200,   // Large desktop
}

// TypeScript module augmentation for type safety
declare module 'react-native-unistyles' {
  export interface UnistylesThemes extends AppThemes {}
  export interface UnistylesBreakpoints extends AppBreakpoints {}
}

// One-time configuration (executed when module is imported)
StyleSheet.configure({
  themes: appThemes,
  breakpoints,
  settings: {
    initialTheme: 'dark',
    adaptiveThemes: false, // Disabled to avoid async issues
  }
})
```

**Configuration Options:**
- `themes`: Object containing all available themes
- `breakpoints`: Responsive breakpoint definitions
- `initialTheme`: Default theme on app startup
- `adaptiveThemes`: Whether to adapt to system theme (disabled for consistency)

### 4. Custom Expo Plugin

**File: `app.plugin.js`**

```javascript
module.exports = require('./plugin/withIosDeploymentTarget');
```

**File: `plugin/withIosDeploymentTarget.js`**

This custom Expo config plugin ensures iOS deployment target is set to 16.0, which is required for optimal Unistyles performance on iOS.

```javascript
const { withDangerousMod } = require('@expo/config-plugins');

const withIosDeploymentTarget = (config) => {
  return withDangerousMod(config, [
    'ios',
    async (config) => {
      // Modifies Podfile to set deployment target to iOS 16.0
      // This ensures compatibility with Unistyles native modules
    }
  ]);
}
```

**File: `app.json`**

```json
{
  "expo": {
    "plugins": [
      // ... other plugins
      "./app.plugin.js"  // Custom plugin for iOS deployment target
    ]
  }
}
```

## Usage Patterns

### Best Practices (Hookless Approach) ✅

The recommended approach is to use `StyleSheet.create()` directly without hooks for optimal performance:

```typescript
import { StyleSheet } from 'react-native-unistyles'
import { View, Text, TouchableOpacity } from 'react-native'

// Create styles with theme access
const styles = StyleSheet.create(theme => ({
  container: {
    backgroundColor: theme.colors.background,
    padding: theme.spacing.md,
    borderRadius: theme.borderRadius.lg,
    // Responsive styles using breakpoints
    width: {
      xs: '100%',      // Mobile
      md: '50%',       // Tablet
      lg: '33.33%',    // Desktop
    },
    // Variants for conditional styling
    variants: {
      elevated: {
        true: {
          shadowColor: '#000',
          shadowOpacity: 0.1,
          elevation: 4,
        },
        false: {
          elevation: 0,
        },
      },
      size: {
        small: { padding: theme.spacing.sm },
        medium: { padding: theme.spacing.md },
        large: { padding: theme.spacing.lg },
      }
    },
  },
  title: {
    ...theme.typography.h1,
    color: theme.colors.textPrimary,
  },
}))

export const MyComponent = ({ elevated = false, size = 'medium' }) => {
  // Apply variants directly (no hooks!)
  styles.useVariants({
    elevated,
    size,
  })
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello World</Text>
    </View>
  )
}
```

### Anti-Pattern (Hook Approach) ❌

Avoid using the `useStyles` hook as it causes unnecessary re-renders:

```typescript
// DON'T DO THIS!
import { useStyles } from '@/src/hooks/useStyles'

export const BadComponent = () => {
  const { styles, theme } = useStyles(stylesheet) // Causes re-renders!
  return <View style={styles.container}>...</View>
}
```

## Design System Structure

### Color System
**File: `src/styles/colors.ts`**

```typescript
export const colors = {
  primary: {
    indigo900: '#312e81',    // Background
    purple600: '#9333ea',    // Primary Purple
    pink500: '#ec4899',      // Primary Pink
  },
  secondary: { /* ... */ },
  accent: { /* ... */ },
  status: { /* ... */ },
  text: { /* ... */ },
  gradients: { /* ... */ },
}
```

### Spacing System
```typescript
export const spacing = {
  xs: 4,
  sm: 8,
  md: 16,
  lg: 24,
  xl: 32,
  xxl: 48,
}
```

### Typography System
```typescript
export const typography = {
  h1: { fontSize: 32, fontWeight: '700', letterSpacing: -0.5 },
  h2: { fontSize: 24, fontWeight: '700', letterSpacing: -0.3 },
  h3: { fontSize: 20, fontWeight: '600' },
  body: { fontSize: 16, fontWeight: '400', lineHeight: 24 },
  // ... more variants
}
```

## Performance Considerations

1. **Compile-Time Optimization**: The Babel plugin processes styles at build time, reducing runtime overhead.

2. **No Re-renders**: Using `StyleSheet.create()` directly (without hooks) prevents unnecessary component re-renders.

3. **Adaptive Themes Disabled**: Setting `adaptiveThemes: false` prevents async theme changes that could cause performance issues.

4. **Singleton Pattern**: Unistyles configuration happens once at app initialization, ensuring consistent performance.

## TypeScript Support

Unistyles provides full TypeScript support through module augmentation:

```typescript
// Type-safe theme access
declare module 'react-native-unistyles' {
  export interface UnistylesThemes extends AppThemes {}
  export interface UnistylesBreakpoints extends AppBreakpoints {}
}

// Helper types for theme values
export type ThemeColors = { /* ... */ }
export type ThemeSpacing = { /* ... */ }
export type ThemeTypography = { /* ... */ }
```

## Migration Guide

### From React Native StyleSheet

```typescript
// Before (React Native StyleSheet)
const styles = StyleSheet.create({
  container: {
    backgroundColor: '#312e81',
    padding: 16,
  }
})

// After (Unistyles)
const styles = StyleSheet.create(theme => ({
  container: {
    backgroundColor: theme.colors.background,
    padding: theme.spacing.md,
  }
}))
```

### From Styled Components or Emotion

```typescript
// Before (Styled Components)
const Container = styled.View`
  background-color: ${props => props.theme.colors.background};
  padding: ${props => props.theme.spacing.md}px;
`

// After (Unistyles)
const styles = StyleSheet.create(theme => ({
  container: {
    backgroundColor: theme.colors.background,
    padding: theme.spacing.md,
  }
}))
```

## Responsive Design

Unistyles supports responsive styles through breakpoints:

```typescript
const styles = StyleSheet.create(theme => ({
  container: {
    // Different values for different screen sizes
    padding: {
      xs: theme.spacing.sm,  // Mobile
      md: theme.spacing.md,  // Tablet
      lg: theme.spacing.lg,  // Desktop
    },
    flexDirection: {
      xs: 'column',          // Stack on mobile
      md: 'row',            // Side-by-side on tablet+
    },
  }
}))
```

## Common Patterns

### Dynamic Styles with Variants

```typescript
const styles = StyleSheet.create(theme => ({
  button: {
    padding: theme.spacing.md,
    borderRadius: theme.borderRadius.md,
    variants: {
      variant: {
        primary: {
          backgroundColor: theme.colors.primary,
        },
        secondary: {
          backgroundColor: theme.colors.secondary,
        },
        ghost: {
          backgroundColor: 'transparent',
          borderWidth: 1,
          borderColor: theme.colors.primary,
        },
      },
      disabled: {
        true: {
          opacity: 0.5,
        },
      },
    },
  },
}))

// Usage
styles.useVariants({
  variant: 'primary',
  disabled: isDisabled,
})
```

### Accessing Theme in Components

For cases where you need theme values outside of styles:

```typescript
import { UnistylesRuntime } from 'react-native-unistyles'

// Get current theme
const currentTheme = UnistylesRuntime.theme

// Get current theme name
const themeName = UnistylesRuntime.themeName

// Change theme
UnistylesRuntime.setTheme('light')
```

## Troubleshooting

### Common Issues

1. **Styles not updating**: Ensure the Babel plugin is configured correctly and restart Metro bundler after changes.

2. **TypeScript errors**: Make sure module augmentation is properly set up in `unistyles.ts`.

3. **iOS build failures**: Verify iOS deployment target is set to 16.0 via the custom Expo plugin.

4. **Theme not applying**: Check that `unistyles-init` is imported as the first import in your root layout.

### Debug Tips

```typescript
// Log current configuration
console.log('Current theme:', UnistylesRuntime.themeName)
console.log('Breakpoint:', UnistylesRuntime.breakpoint)
console.log('Screen dimensions:', UnistylesRuntime.screen)
```

## Resources

- [React Native Unistyles Documentation](https://unistyl.es)
- [Expo Config Plugins](https://docs.expo.dev/config-plugins/introduction/)
- [TypeScript Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation)