## Security Design Improvements

### 1. Authentication Flow

#### Implement Logout Functionality

**Issue:**

The provided `AccountController` lacks a Logout endpoint.  
A secure logout mechanism requires immediate token invalidation to prevent unauthorized access if a token is compromised or a user explicitly logs out.

**Fix Action:**

- Add a **Logout endpoint** to the `AccountController`.
- This endpoint should receive the **JWT or a session identifier**.
- Invalidate the session associated with the JWT.

---

### 2. Database Design

#### Add CreatedAt Timestamp to ApplicationUser

**Issue:**

The `ApplicationUser` class does not explicitly track a `CreatedAt` timestamp.

**Fix Action:**

- Modify the `ApplicationUser` class to include a `CreatedAt` property.
- Generate and apply a new **database migration** to add this column to the `AspNetUsers` table.

---

### 3. JWT Structure

#### Include Email Claim in JWT

**Issue:**

The generated JWTs do not explicitly include the user's email address as an email claim.  
While `ClaimTypes.Name` might contain the email, using a dedicated email claim is clearer and more standard-compliant.

**Fix Action:**

Modify the `GenerateToken` helper method in `AccountController`.

Add the claim:

```
JwtRegisteredClaimNames.Email
```

using:

```
user.Email
```

---

### 4. Authentication & Authorization Middleware

#### a) Authentication Middleware

**Issue:**

The provided `AccountController` is responsible for issuing JWTs, but validating incoming JWTs and enforcing application-specific security rules (such as session activity and user account status) is not present within the controller.

**Fix Action:**

Configure custom validation logic within the `OnTokenValidated` event of the JWT Bearer authentication handler in `Program.cs`.

Custom logic should ensure:

**Session Activity Check**

- Retrieve the `sid` claim from the JWT.
- Query the `UserSessions` table to ensure that a session with the matching:
  - `SessionId`
  - `UserId`
- Exists and:
  - `IsActive = true`
  - `ExpiresAt` has not passed.

**User Account Activity Check**

- Retrieve the `UserId` (from `ClaimTypes.NameIdentifier`) from the JWT.
- Query the `Users` table (or use `UserManager`) to ensure:
  - The `ApplicationUser` exists.
  - `IsActive = true`.

---

#### b) Authorization Middleware

**Issue:**

The entire authorization middleware is missing.

**Fix Action:**

Apply similar validation steps used in authentication middleware.

The middleware should ensure:

- The request contains a **valid JWT**.
- The **token signature is valid**.
- The **token has not expired**.
- The **session is still active**.
- The **user has the required role**.
- The **user account is active**.

---

### 5. Security Improvements

Aside from **password hashing**, additional security measures are currently missing.

Recommended improvements include:

- Rate limiting for authentication endpoints.
- Brute-force protection.
- Account lockout after multiple failed login attempts.
- Secure logging and monitoring.
- Strong password policies.
- Token expiration and refresh token mechanisms.
