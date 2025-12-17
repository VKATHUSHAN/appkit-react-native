# AppKit React Native - AI Coding Agent Instructions

## Project Overview

AppKit is a React Native SDK for web3 wallet connectivity and blockchain interactions. This is a **monorepo** managed with Yarn workspaces and Turborepo, providing multiple integration paths (Wagmi, Ethers, SIWE) for web3 dapp development.

## Architecture

### Monorepo Structure

```
packages/
  core/          # Controllers, utilities, core state management (Valtio proxies)
  ui/            # Reusable UI components (Text, Button, List, etc.)
  common/        # Shared types, constants, utilities
  scaffold/      # Main modal components, hooks, high-level APIs
  scaffold-utils/# Ethers/Wagmi-specific utilities
  wagmi/         # Wagmi-specific client & connectors
  ethers/        # Ethers v6 adapter
  ethers5/       # Ethers v5 adapter
  siwe/          # Sign-In with Ethereum controller
  wallet/        # Wallet-specific logic
  auth-wagmi/    # Auth features for Wagmi
  coinbase-wagmi/# Coinbase integration for Wagmi
  cli/           # CLI tools

apps/
  gallery/       # Storybook-like gallery for UI components
  native/        # Expo-based example app for testing
```

### State Management Pattern

**Critical**: All controllers use **Valtio proxy-based reactivity**. Do NOT use Redux, Context API, or other state solutions for controller state.

```typescript
// Example: packages/core/src/controllers/AccountController.ts
import { proxy } from 'valtio';
import { subscribeKey as subKey } from 'valtio/utils';

const state = proxy<AccountControllerState>({
  isConnected: false,
  address: undefined
});

export const AccountController = {
  state,
  subscribeKey<K extends StateKey>(key: K, callback: (value: AccountControllerState[K]) => void) {
    return subKey(state, key, callback);
  },
  setIsConnected(isConnected: boolean) {
    state.isConnected = isConnected;
  }
};
```

### Navigation & Routing

**Do NOT use `react-navigation`**. Use the internal `RouterController` from `@reown/appkit-core-react-native`:

```typescript
import { RouterController } from '@reown/appkit-core-react-native';

// Navigate programmatically
RouterController.push('Account');
RouterController.goBack();
```

See [packages/core/src/controllers/RouterController.ts](../packages/core/src/controllers/RouterController.ts) for all available views.

## Development Workflows

### Setup & Installation

```bash
# Install dependencies (uses Yarn 4.0.2)
yarn

# Run development gallery (Storybook-like UI component viewer)
yarn gallery

# Run native example app
yarn ios           # iOS simulator
yarn android       # Android emulator
yarn web           # Web browser
```

### Building & Testing

```bash
# Build all packages (uses Turborepo with dependency graph)
yarn build

# Run tests (builds first, then runs Jest tests)
yarn test

# Lint & format
yarn lint
yarn prettier
yarn format        # Auto-fix formatting issues

# Run Playwright E2E tests (in apps/native)
yarn playwright:test
```

### Pre-commit Hooks

Lefthook runs automatically before commits:
- ESLint on `.{js,ts,jsx,tsx}` files
- TypeScript type checking (`tsc --noEmit`)
- Commitlint enforces [Conventional Commits](https://www.conventionalcommits.org/)

Valid prefixes: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`

### Publishing & Versioning

Uses **Changesets** for version management:

```bash
# Create a changeset (tracks changes for next release)
yarn changeset

# Version packages (bumps versions based on changesets)
yarn changeset:version

# Publish to npm
yarn changeset:publish
```

The `version:update` script syncs versions across packages via `scripts/bump-version.sh`.

## Code Conventions

### Component Architecture

**Function-based components only**. Use hooks (`useState`, `useEffect`, `useCallback`, `useMemo`, `React.memo`) for all logic.

```tsx
import React, { useCallback } from 'react';
import { Text } from '@reown/appkit-ui-react-native';

export const MyComponent = React.memo(({ onPress }) => {
  const handlePress = useCallback(() => {
    onPress();
  }, [onPress]);

  return <Text onPress={handlePress}>Click me</Text>;
});
```

### UI Component Preferences

**Always use `@reown/appkit-ui-react-native` components** instead of React Native defaults:

```tsx
// ✅ Correct
import { Text, Button } from '@reown/appkit-ui-react-native';

// ❌ Avoid
import { Text, TouchableOpacity } from 'react-native';
```

Exception: Use React Native's `FlatList` directly for list rendering (do NOT wrap in custom `<List />`).

### Import Organization

1. **External libraries** (`react`, `valtio`, `viem`, `react-native`)
2. **Internal SDK packages** (`@reown/appkit-*-react-native`)
3. **Relative imports** (`./controllers/*`, `./utils/*`)

```typescript
import React from 'react';
import { proxy } from 'valtio';
import { Text } from '@reown/appkit-ui-react-native';
import { AccountController } from './controllers/AccountController';
```

### TypeScript Strictness

Follow `tsconfig.json` settings:
- `"strict": true`
- `"noUncheckedIndexedAccess": true`
- Always define types for controller states, function parameters, and component props

### Testing Strategy

- **Shared Jest setup**: All packages import from `@shared-jest-setup` (see [TESTING.md](../TESTING.md))
- **80%+ coverage target** for controllers and utilities
- Use `@testing-library/react-native` for component testing
- Mock `AsyncStorage`, `react-native-svg`, and other RN modules via shared setup

Example test structure:

```typescript
import { render } from '@testing-library/react-native';
import { MyComponent } from './MyComponent';

describe('MyComponent', () => {
  it('renders correctly', () => {
    const { getByText } = render(<MyComponent />);
    expect(getByText('Expected Text')).toBeTruthy();
  });
});
```

## Integration Patterns

### Controllers as Single Source of Truth

Controllers in `packages/core/src/controllers/` manage all state:
- `AccountController` - User account state (address, balance, connection status)
- `NetworkController` - Network/chain management
- `ModalController` - Modal open/close state
- `RouterController` - Navigation state
- `ConnectionController` - Wallet connection logic
- `ThemeController` - Theme mode and variables

**Never duplicate state** in components. Always subscribe to controller state via `useSnapshot` (Valtio) or direct subscription.

### Client Initialization

The `AppKitScaffold` class (in `packages/scaffold/src/client.ts`) orchestrates all controllers. Integration packages (wagmi/ethers) wrap this scaffold:

```typescript
// Typical integration pattern (see packages/wagmi or packages/ethers)
import { AppKitScaffold } from '@reown/appkit-scaffold-react-native';

export function createAppKit(config: AppKitConfig) {
  return new AppKitScaffold({
    projectId: config.projectId,
    metadata: config.metadata,
    networkControllerClient: networkClient,
    connectionControllerClient: connectionClient,
    // ... other clients
  });
}
```

### Error Handling

Use `ErrorUtil` from `@reown/appkit-common-react-native` for consistent error formatting:

```typescript
import { ErrorUtil } from '@reown/appkit-common-react-native';

async function connectWallet() {
  try {
    // connection logic
  } catch (error) {
    throw ErrorUtil.formatError(error, 'Wallet connection failed');
  }
}
```

Always wrap async operations in `try-catch`. Fail gracefully with descriptive error messages.

## Performance Optimizations

- **Memoize expensive renders**: Use `React.memo`, `useCallback`, `useMemo`
- **FlatList for lists**: Use `keyExtractor` and avoid `.map()` for large datasets
- **Native animations**: Prefer React Native's `Animated` API over third-party libraries
- **Debounce API calls**: Use `lodash.debounce` for search inputs, network requests

## Key Conventions

- **AsyncStorage**: Only for non-sensitive data (user preferences, session state). Never store private keys or auth tokens.
- **No inline styles**: Use component-level styling or theme variables from `ThemeController`.
- **JSDoc comments**: Document all public APIs (`@param`, `@returns`, `@throws`).
- **Package exports**: Each package has an `index.ts` that explicitly exports public APIs. Do not export internal utilities.

## Common Tasks

### Adding a New Controller

1. Create `packages/core/src/controllers/MyController.ts`
2. Define state interface and initialize with `proxy<MyControllerState>(...)`
3. Export controller in `packages/core/src/index.ts`
4. Add unit tests in `packages/core/src/__tests__/MyController.test.ts`

### Adding a New UI Component

1. Create `packages/ui/src/components/my-component/index.tsx`
2. Use existing UI primitives from `@reown/appkit-ui-react-native`
3. Export in `packages/ui/src/index.ts`
4. Add to gallery: `apps/gallery/stories/my-component.stories.tsx`

### Modifying Build Configuration

- **Turborepo tasks**: Edit `turbo.json` (defines task dependencies and outputs)
- **Package builds**: Most packages use `react-native-builder-bob` (see `bob.config.js` in each package)
- **Monorepo dependencies**: Update `package.json` workspaces array if adding new packages

## External Dependencies

- **WalletConnect**: Core dependency for wallet connection protocol
- **Viem**: Blockchain utilities (address formatting, contract interactions)
- **Wagmi**: React hooks for Ethereum (used in wagmi integration)
- **Expo**: Used in example app (`apps/native`)
- **React Native SVG**: For icon rendering
- **AsyncStorage**: For persistence

## Documentation & Resources

- **Official Docs**: https://docs.reown.com/appkit/react-native/core/installation
- **Examples**: https://github.com/reown-com/react-native-examples/
- **License**: Reown AppKit Community License (see [LICENSE.md](../LICENSE.md))

## When in Doubt

1. Check existing controller implementations in `packages/core/src/controllers/`
2. Reference UI component patterns in `packages/ui/src/components/`
3. Look at integration examples in `packages/wagmi/` or `packages/ethers/`
4. Review shared test setup in `jest-shared-setup.ts` and [TESTING.md](../TESTING.md)
