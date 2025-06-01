# Multi-Factor Authentication (MFA)

Second authentication factor using TOTP (authenticator apps), SMS, or backup codes.

## Why It Matters

- Protects against password compromise (something you know + something you have)
- Required for admin accounts and sensitive operations in most compliance frameworks
- TOTP is free, widely supported, and does not depend on SMS delivery

## Key Recommendations

- **Packages**: `OtpNet` for TOTP generation/validation, `QRCoder` for QR codes
- **TOTP flow**: Generate secret, show QR URI, verify code before enabling

```csharp
// Generate
var secret = mfaService.GenerateSecret(); // 20 random bytes, Base32-encoded
var uri = $"otpauth://totp/{issuer}:{email}?secret={secret}&issuer={issuer}";

// Validate (allow +/- 1 time-step for clock skew)
var totp = new Totp(Base32Encoding.ToBytes(secret));
return totp.VerifyTotp(code, out _, new VerificationWindow(1, 1));
```

- **Login flow with MFA**: Password check returns `MfaRequired: true` + short-lived MFA token (5 min JWT), then `/auth/mfa/verify` exchanges MFA token + TOTP code for access/refresh tokens
- **Backup codes**: Generate 8-10 codes, hash (SHA256) before storing, remove after single use, allow regeneration
- **Remember device**: Store trusted device fingerprint for 30 days to skip MFA on known devices
- **SMS fallback**: Use as secondary option (Twilio); 6-digit code, 5-minute expiry
- **Disable flow**: Require valid TOTP or backup code to disable MFA (prevent unauthorized disabling)

## Enrollment Checklist

1. User requests MFA enable -- generate secret + backup codes
2. Display QR code + backup codes (only time codes are shown in plain text)
3. User scans QR, submits TOTP code to verify
4. Mark MFA as verified and active

## Pitfalls to Avoid

- Enabling MFA before the user verifies a valid TOTP code
- Storing backup codes in plaintext (hash them)
- Not rate-limiting MFA code verification attempts
- Using `System.Random` for SMS codes (use `RandomNumberGenerator`)
- Hardcoding the MFA token signing key (store in secrets manager)
- Forgetting a recovery path when users lose their authenticator device
