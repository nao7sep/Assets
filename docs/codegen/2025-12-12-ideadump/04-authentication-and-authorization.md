# Authentication and Authorization

## Authentication Flow

### User Registration

```
┌─────────┐         ┌──────────────┐         ┌──────────┐
│ Browser │         │ Blazor Server│         │ Database │
└────┬────┘         └──────┬───────┘         └─────┬────┘
     │                     │                       │
     │ Submit Registration │                       │
     │────────────────────>│                       │
     │                     │ Validate Input        │
     │                     │──────────┐            │
     │                     │          │            │
     │                     │<─────────┘            │
     │                     │                       │
     │                     │ Create User           │
     │                     │──────────────────────>│
     │                     │                       │
     │                     │ Generate Token        │
     │                     │<──────────────────────│
     │                     │                       │
     │                     │ Send Confirmation Email
     │                     │───────────────────┐   │
     │                     │                   │   │
     │<────────────────────│                   │   │
     │ "Check your email"  │                   │   │
     │                     │                   ▼   │
     │                     │            ┌──────────┴──┐
     │                     │            │ Email Sent  │
     │                     │            └─────────────┘
```

#### Registration Process

1. **User submits registration form**:
   - Email address
   - Password (min 8 chars, must include uppercase, lowercase, digit, special char)
   - Optional: Display name

2. **Server validates input**:
   - Email format validation
   - Email uniqueness check
   - Password strength validation

3. **User record created**:
   ```csharp
   var user = new ApplicationUser
   {
       Id = Guid.NewGuid(),
       UserName = email,
       Email = email,
       DisplayName = displayName,
       EmailConfirmed = false,
       DailyIdeaLimit = defaultLimit, // From SystemConfiguration
       CreatedAt = DateTime.UtcNow
   };

   var result = await _userManager.CreateAsync(user, password);
   ```

4. **Confirmation email sent**:
   - Token generated via ASP.NET Core Identity
   - Email contains link: `https://ideadump.example.com/confirm-email?userId={id}&token={token}`
   - Token valid for 24 hours

5. **User clicks confirmation link**:
   ```csharp
   var result = await _userManager.ConfirmEmailAsync(user, token);
   if (result.Succeeded)
   {
       user.EmailConfirmed = true;
       // User can now log in
   }
   ```

### Login Flow

```csharp
var result = await _signInManager.PasswordSignInAsync(
    email,
    password,
    isPersistent: true,
    lockoutOnFailure: true
);

if (result.Succeeded)
{
    user.LastLoginAt = DateTime.UtcNow;
    await _context.SaveChangesAsync();
    // Redirect to dashboard
}
else if (result.IsLockedOut)
{
    // Show lockout message
}
else if (result.RequiresTwoFactor)
{
    // Not implemented in v0.1
}
else
{
    // Invalid credentials
}
```

### Password Reset Flow

1. **User requests password reset**:
   - Submits email address
   - Server generates reset token
   - Email sent with reset link

2. **User clicks reset link**:
   - Navigates to `/reset-password?userId={id}&token={token}`
   - Submits new password

3. **Server resets password**:
   ```csharp
   var result = await _userManager.ResetPasswordAsync(user, token, newPassword);
   ```

## Authorization

### Role-Based Access Control

Two roles defined:
1. **User** (default)
2. **Admin** (assigned via `ApplicationUser.IsAdmin = true`)

### Authorization Policies

```csharp
// Program.cs
builder.Services.AddAuthorizationCore(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireAssertion(context =>
        {
            var userIdClaim = context.User.FindFirst(ClaimTypes.NameIdentifier);
            if (userIdClaim == null) return false;

            var userId = Guid.Parse(userIdClaim.Value);
            var user = dbContext.Users.Find(userId);
            return user?.IsAdmin == true;
        }));
});
```

### Page-Level Authorization

```csharp
@page "/admin"
@attribute [Authorize(Policy = "AdminOnly")]

<h1>Admin Dashboard</h1>
<!-- Admin content -->
```

### Resource-Level Authorization

Users can only access their own resources:

```csharp
public async Task<Channel> GetChannelAsync(Guid channelId, Guid userId)
{
    var channel = await _context.Channels
        .Include(c => c.Recipients)
        .FirstOrDefaultAsync(c => c.Id == channelId && c.UserId == userId);

    if (channel == null)
        throw new UnauthorizedAccessException("Channel not found or access denied");

    return channel;
}
```

### API Key Authorization

API keys are used for external AI services, not for user authentication.

- **System API key**: Stored in `SystemConfiguration` table
- **User API key**: Optional override in `ApplicationUser.ApiKey`
- **Channel API key**: Optional override in `Channel.ApiKey`

Keys are encrypted at rest and never exposed to client.

## Security Policies

### Password Policy

Configured via ASP.NET Core Identity:

```csharp
builder.Services.Configure<IdentityOptions>(options =>
{
    // Password settings
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
    options.Password.RequiredUniqueChars = 4;

    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    // User settings
    options.User.RequireUniqueEmail = true;

    // Sign-in settings
    options.SignIn.RequireConfirmedEmail = true;
});
```

### Session Management

```csharp
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always; // HTTPS only
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.ExpireTimeSpan = TimeSpan.FromDays(14);
    options.SlidingExpiration = true;

    options.LoginPath = "/login";
    options.LogoutPath = "/logout";
    options.AccessDeniedPath = "/access-denied";
});
```

### HTTPS Enforcement

```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}
```

### CORS (Not Needed)

Since IdeaDump is Blazor Server (not Blazor WebAssembly or API), CORS configuration is not required.

## Data Protection

### API Key Encryption

```csharp
public class SecureConfigurationService
{
    private readonly IDataProtectionProvider _dataProtection;
    private readonly IDataProtector _protector;

    public SecureConfigurationService(IDataProtectionProvider dataProtection)
    {
        _dataProtection = dataProtection;
        _protector = dataProtection.CreateProtector("IdeaDump.ApiKeys");
    }

    public string EncryptApiKey(string plainText)
    {
        return _protector.Protect(plainText);
    }

    public string DecryptApiKey(string cipherText)
    {
        return _protector.Unprotect(cipherText);
    }
}
```

### SMTP Password Encryption

Same approach as API keys:

```csharp
var protector = _dataProtection.CreateProtector("IdeaDump.SmtpPasswords");
channel.SmtpPassword = protector.Protect(plainTextPassword);
```

### Database Encryption

SQLite database file can be encrypted using:
- **SQLCipher** extension (optional)
- **File-system encryption** (Azure App Service default)

For v0.1, rely on Azure's file-system encryption.

## Consent Management

### Recipient Confirmation Flow

Recipients must confirm twice:
1. **System-wide consent**: Accept to receive emails from IdeaDump
2. **Channel-specific consent**: Accept to receive emails from specific channel

```
┌───────────────┐         ┌──────────────┐         ┌──────────┐
│   Recipient   │         │ Blazor Server│         │ Database │
└───────┬───────┘         └──────┬───────┘         └─────┬────┘
        │                        │                       │
        │ (User adds recipient)  │                       │
        │                        │ Create ChannelRecipient
        │                        │──────────────────────>│
        │                        │                       │
        │                        │ Generate Token        │
        │                        │<──────────────────────│
        │                        │                       │
        │<───────────────────────│ Send Confirmation     │
        │ "Click to confirm"     │                       │
        │                        │                       │
        │ Click Confirmation Link│                       │
        │───────────────────────>│                       │
        │                        │ Verify Token          │
        │                        │──────────────────────>│
        │                        │                       │
        │                        │ Mark Confirmed        │
        │                        │<──────────────────────│
        │                        │                       │
        │<───────────────────────│                       │
        │ "Subscription confirmed"                       │
```

### Unsubscribe Flow

Every email includes an unsubscribe link:
- URL: `https://ideadump.example.com/unsubscribe?token={unsubscribeToken}`
- One-click unsubscribe (no login required)
- Sets `ChannelRecipient.IsUnsubscribed = true`

```csharp
public async Task UnsubscribeAsync(string token)
{
    var recipient = await _context.ChannelRecipients
        .FirstOrDefaultAsync(r => r.UnsubscribeToken == token);

    if (recipient == null)
        throw new InvalidOperationException("Invalid unsubscribe token");

    recipient.IsUnsubscribed = true;
    recipient.UnsubscribedAt = DateTime.UtcNow;
    await _context.SaveChangesAsync();

    // Optional: Send confirmation email
}
```

## Account Deletion

User can delete their account:

1. **User initiates deletion**:
   - Navigates to account settings
   - Clicks "Delete Account"
   - Confirms password

2. **Server deletes cascade**:
   ```csharp
   public async Task DeleteAccountAsync(Guid userId)
   {
       var user = await _context.Users
           .Include(u => u.Channels)
               .ThenInclude(c => c.Ideas)
           .Include(u => u.Channels)
               .ThenInclude(c => c.Recipients)
           .FirstOrDefaultAsync(u => u.Id == userId);

       if (user == null)
           throw new InvalidOperationException("User not found");

       // Cascade delete: Channels → Ideas → Recipients
       _context.Users.Remove(user);
       await _context.SaveChangesAsync();

       await _signInManager.SignOutAsync();
   }
   ```

3. **What gets deleted**:
   - User record
   - All channels owned by user
   - All ideas in those channels
   - All recipients in those channels
   - Processing logs for those ideas

4. **What persists**:
   - System logs (anonymized: `UserId` kept for audit trail)

## Multi-Tenancy Considerations

IdeaDump is **single-tenant** (one deployment per instance). Each user's data is isolated by `UserId` foreign keys.

No shared resources between users except:
- System configuration
- System-wide API key (if user doesn't provide their own)

## Authentication Flows (Code Examples)

### Registration Component

```csharp
@page "/register"
@inject UserManager<ApplicationUser> UserManager
@inject IEmailService EmailService

<h1>Register</h1>

<EditForm Model="@model" OnValidSubmit="@HandleRegistration">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <InputText @bind-Value="model.Email" placeholder="Email" />
    <InputText type="password" @bind-Value="model.Password" placeholder="Password" />
    <InputText @bind-Value="model.DisplayName" placeholder="Display Name (optional)" />

    <button type="submit">Register</button>
</EditForm>

@code {
    private RegisterModel model = new();

    private async Task HandleRegistration()
    {
        var user = new ApplicationUser
        {
            UserName = model.Email,
            Email = model.Email,
            DisplayName = model.DisplayName
        };

        var result = await UserManager.CreateAsync(user, model.Password);

        if (result.Succeeded)
        {
            var token = await UserManager.GenerateEmailConfirmationTokenAsync(user);
            var confirmUrl = $"{NavigationManager.BaseUri}confirm-email?userId={user.Id}&token={Uri.EscapeDataString(token)}";

            await EmailService.SendConfirmationEmailAsync(user.Email, confirmUrl);

            // Show success message
        }
        else
        {
            // Show errors
        }
    }

    private class RegisterModel
    {
        [Required, EmailAddress]
        public string Email { get; set; } = "";

        [Required, MinLength(8)]
        public string Password { get; set; } = "";

        public string? DisplayName { get; set; }
    }
}
```

### Login Component

```csharp
@page "/login"
@inject SignInManager<ApplicationUser> SignInManager

<h1>Login</h1>

<EditForm Model="@model" OnValidSubmit="@HandleLogin">
    <InputText @bind-Value="model.Email" placeholder="Email" />
    <InputText type="password" @bind-Value="model.Password" placeholder="Password" />
    <InputCheckbox @bind-Value="model.RememberMe" /> Remember me

    <button type="submit">Login</button>
</EditForm>

@code {
    private LoginModel model = new();

    private async Task HandleLogin()
    {
        var result = await SignInManager.PasswordSignInAsync(
            model.Email,
            model.Password,
            model.RememberMe,
            lockoutOnFailure: true
        );

        if (result.Succeeded)
        {
            NavigationManager.NavigateTo("/dashboard");
        }
        else if (result.IsLockedOut)
        {
            // Show lockout message
        }
        else
        {
            // Show error
        }
    }
}
```

## Security Checklist

- [x] HTTPS enforcement
- [x] Strong password policy
- [x] Email confirmation required
- [x] Account lockout after failed attempts
- [x] Encrypted API keys and passwords
- [x] HttpOnly, Secure, SameSite cookies
- [x] CSRF protection (Blazor built-in)
- [x] XSS protection (Blazor encoding)
- [x] SQL injection protection (EF Core parameterized queries)
- [x] Resource-level authorization
- [x] Recipient consent workflow
- [x] One-click unsubscribe
- [x] Secure session management
- [x] Admin-only pages protected
- [x] Sensitive data redacted from logs

---

**Next Document**: [05-channel-and-recipient-management.md](05-channel-and-recipient-management.md)
