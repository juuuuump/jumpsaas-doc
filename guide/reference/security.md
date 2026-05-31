---
title: "Security Features"
---

# Security Features

This document describes the security features implemented in JumpSaaS, including active sessions management, password changes, and account deletion.

## Overview

The security features provide users with control over their account security through:
- **Active Sessions Management**: View and revoke active login sessions
- **Password Changes**: Update account password with current password verification
- **Account Deletion**: Permanently delete account with confirmation

## Architecture

All features follow the three-layer architecture pattern:
- **Repository Layer**: Direct database operations
- **Service Layer**: Business logic and transformations
- **Actions Layer**: Server Actions for client-server communication

## Active Sessions

### Features
- View all active sessions with device/browser information
- See IP address, user agent, and last active time
- Identify current session with badge
- Revoke individual sessions
- Revoke all other sessions at once

### Components
- **ActiveSessionsCard** (`src/services/auth/ui/active-sessions-card.tsx`): Main card component
- **SessionList** (`src/services/auth/ui/session-list.tsx`): Table displaying sessions
- **useActiveSessions** (`src/services/auth/use-active-sessions.ts`): State management hook

### Implementation
```typescript
// Session types with device info
type SessionInfo = {
  id: string;
  ipAddress: string | null;
  userAgent: string | null;
  createdAt: Date;
  expiresAt: Date;
  isCurrent: boolean;
  device: DeviceInfo;
};

// Device detection using ua-parser-js
function parseUserAgent(userAgent: string | null): DeviceInfo {
  const parser = new UAParser(userAgent);
  return {
    browser: parser.getBrowser().name,
    os: parser.getOS().name,
    deviceType: parser.getDevice().type || "Desktop",
  };
}
```

### Server Actions
- `listSessions()`: Returns all user sessions with device info
- `revokeSession(sessionId)`: Revokes a specific session
- `revokeAllOtherSessions()`: Revokes all sessions except current

### Error Codes
- `UNAUTHORIZED`: User not authenticated
- `INVALID_SESSION_TOKEN`: Attempting to revoke current session
- `REVOKE_FAILED`: Database operation failed
- `NETWORK_ERROR`: Network or unexpected error

## Password Changes

### Features
- Verify current password before changing
- Validate new password (minimum 8 characters)
- Confirm new password matches
- Password visibility toggles
- Real-time validation feedback

### Components
- **ChangePasswordCard** (`src/services/auth/ui/change-password-card.tsx`): Main card component
- **ChangePasswordDialog** (`src/services/auth/ui/change-password-dialog.tsx`): Password change form
- **useChangePassword** (`src/services/auth/use-change-password.ts`): State management hook

### Implementation
```typescript
// Password service with bcrypt
async function changeUserPassword(
  userId: string,
  currentPassword: string,
  newPassword: string
): Promise<void> {
  // 1. Find credential account
  const account = await findCredentialAccount(userId);

  // 2. Verify current password
  const isValid = await verifyPassword(currentPassword, account.password);
  if (!isValid) throw new Error("INVALID_PASSWORD");

  // 3. Hash and update new password
  const hashedPassword = await hashPassword(newPassword);
  await updateAccountPassword(account.id, hashedPassword);
}
```

### Server Actions
- `changePassword(currentPassword, newPassword)`: Changes user password

### Error Codes
- `UNAUTHORIZED`: User not authenticated
- `NO_PASSWORD_ACCOUNT`: No credential account found
- `NO_PASSWORD_SET`: Account has no password (OAuth-only)
- `INVALID_PASSWORD`: Current password incorrect
- `PASSWORD_MISMATCH`: New passwords don't match
- `PASSWORD_TOO_SHORT`: Password less than 8 characters
- `CHANGE_FAILED`: Database operation failed

## Account Deletion

### Features
- Permanent account deletion
- Requires typing "DELETE" to confirm
- Automatic sign out after deletion
- Cascade deletes related data (sessions, subscriptions, etc.)
- Destructive styling to emphasize action

### Components
- **DeleteAccountCard** (`src/services/auth/ui/delete-account-card.tsx`): Main card component
- **DeleteAccountDialog** (`src/services/auth/ui/delete-account-dialog.tsx`): Confirmation dialog
- **useDeleteAccount** (`src/services/auth/use-delete-account.ts`): State management hook

### Implementation
```typescript
// Deletion service
async function deleteUserAccount(userId: string): Promise<void> {
  // Database cascade deletes handle related data
  await db.delete(auth_users).where(eq(auth_users.id, userId));
}

// Action with sign out and redirect
export async function deleteAccount() {
  const session = await auth.api.getSession({ headers: await headers() });

  await deleteUserAccount(session.user.id);
  await auth.api.signOut({ headers: await headers() });

  redirect("/");
}
```

### Server Actions
- `deleteAccount()`: Deletes account, signs out, and redirects

### Error Codes
- `UNAUTHORIZED`: User not authenticated
- `DELETE_FAILED`: Database operation failed

### Database Cascade Deletes
The following are automatically deleted when a user is deleted:
- Sessions (`auth_session`)
- Accounts (`auth_account`)
- Two-factor settings (`two_factor`)
- Subscriptions (`subscription`)
- Payment methods (`payment_method`)
- Invoices (`invoice`)

## Internationalization

All features support English and German translations:
- Error messages
- Success messages
- Form labels
- Dialog content
- Button text

Translation keys are located in:
- `messages/en/settings.json`
- `messages/de/settings.json`

## Security Considerations

### Password Hashing
- Uses bcryptjs with 10 rounds
- Passwords never stored or transmitted in plain text
- Current password verified before changes

### Session Security
- Users cannot revoke their current session
- Session revocation is immediate
- No orphaned sessions remain after user deletion

### Account Deletion
- Requires explicit confirmation ("DELETE" text)
- Irreversible action
- Automatic cleanup of all user data
- User signed out and redirected

## Testing

All features have been tested for:
- [YOURS] TypeScript type safety (no errors)
- [YOURS] Build compilation success
- [YOURS] Translation coverage (EN + DE)
- [YOURS] Proper error handling
- [YOURS] Server action patterns match codebase conventions

## Usage

The security features are available at `/settings/security` and include:

1. **Two-Factor Authentication** (existing)
2. **Active Sessions** (new)
3. **Change Password** (new)
4. **Delete Account** (new)

All features follow the same UI patterns:
- Card-based layout
- Icon with duotone weight
- Primary/10 background for icons
- Consistent spacing and styling
- Toast notifications for feedback
