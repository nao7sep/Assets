# Authentication and OAuth2 Implementation

## Overview

The application supports two authentication methods:
1. **Password authentication**: Traditional username/password for standard IMAP servers
2. **OAuth2 authentication**: Required for Gmail and other OAuth-enabled providers

## Authentication Architecture

```
┌─────────────────────────────────────────────────────────┐
│              IAuthenticator Interface                   │
└─────────────────────────────────────────────────────────┘
                          ▲
                          │
          ┌───────────────┴───────────────┐
          │                               │
┌─────────────────────┐      ┌────────────────────────┐
│ Password            │      │ OAuth2                 │
│ Authenticator       │      │ Authenticator          │
└─────────────────────┘      └────────────────────────┘
                                      │
                                      │ uses
                                      ▼
                          ┌────────────────────────┐
                          │ IOAuth2Provider        │
                          └────────────────────────┘
                                      ▲
                                      │
                          ┌───────────┴────────────┐
                          │                        │
                  ┌───────────────┐      ┌─────────────────┐
                  │ Gmail         │      │ Outlook         │
                  │ OAuth2Provider│      │ OAuth2Provider  │
                  └───────────────┘      └─────────────────┘
```

## Interfaces

### IAuthenticator

```csharp
public interface IAuthenticator
{
    /// <summary>
    /// Authenticate an IMAP client with the provided account configuration
    /// </summary>
    Task AuthenticateAsync(
        IImapClient client,
        AccountConfig account,
        CancellationToken ct = default);
}
```

### IOAuth2Provider

```csharp
public interface IOAuth2Provider
{
    /// <summary>
    /// Get the OAuth2 provider name (e.g., "Google", "Microsoft")
    /// </summary>
    string ProviderName { get; }

    /// <summary>
    /// Generate the authorization URL for user consent
    /// </summary>
    string GetAuthorizationUrl(string redirectUri, string[] scopes);

    /// <summary>
    /// Exchange authorization code for access and refresh tokens
    /// </summary>
    Task<TokenResponse> ExchangeCodeForTokensAsync(
        string code,
        string redirectUri,
        CancellationToken ct = default);

    /// <summary>
    /// Refresh an expired access token using a refresh token
    /// </summary>
    Task<TokenResponse> RefreshAccessTokenAsync(
        string refreshToken,
        CancellationToken ct = default);
}
```

## Password Authentication

### Implementation

```csharp
public class PasswordAuthenticator : IAuthenticator
{
    private readonly ILogger<PasswordAuthenticator> _logger;

    public PasswordAuthenticator(ILogger<PasswordAuthenticator> logger)
    {
        _logger = logger;
    }

    public async Task AuthenticateAsync(
        IImapClient client,
        AccountConfig account,
        CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(account.Password))
            throw new InvalidOperationException(
                $"Password is required for account '{account.Name}'");

        _logger.LogInformation(
            "Authenticating {Account} with password",
            account.Name);

        await client.AuthenticateAsync(
            account.Username,
            account.Password,
            ct);

        _logger.LogInformation(
            "Successfully authenticated {Account}",
            account.Name);
    }
}
```

### Configuration Example

```json
{
  "name": "personal",
  "server": "imap.example.com",
  "username": "user@example.com",
  "authType": "password",
  "password": "${PERSONAL_PASSWORD}"
}
```

## OAuth2 Authentication

### Gmail OAuth2 Provider

#### Google Cloud Console Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project (or select existing)
3. Enable **Gmail API**
4. Create OAuth 2.0 credentials:
   - Application type: **Desktop app**
   - Note the **Client ID** and **Client Secret**
5. Add authorized redirect URI: `http://localhost:8080`

#### Required OAuth2 Scopes

```
https://mail.google.com/
```

This scope provides full access to Gmail via IMAP/SMTP.

#### Implementation

```csharp
public class GmailOAuth2Provider : IOAuth2Provider
{
    private readonly string _clientId;
    private readonly string _clientSecret;
    private readonly ILogger<GmailOAuth2Provider> _logger;
    private readonly HttpClient _httpClient;

    private const string AuthEndpoint = "https://accounts.google.com/o/oauth2/v2/auth";
    private const string TokenEndpoint = "https://oauth2.googleapis.com/token";
    private const string Scope = "https://mail.google.com/";

    public string ProviderName => "Google";

    public GmailOAuth2Provider(
        string clientId,
        string clientSecret,
        ILogger<GmailOAuth2Provider> logger,
        HttpClient? httpClient = null)
    {
        _clientId = clientId;
        _clientSecret = clientSecret;
        _logger = logger;
        _httpClient = httpClient ?? new HttpClient();
    }

    public string GetAuthorizationUrl(string redirectUri, string[] scopes)
    {
        var scope = string.Join(" ", scopes.Any() ? scopes : new[] { Scope });

        var parameters = new Dictionary<string, string>
        {
            ["client_id"] = _clientId,
            ["redirect_uri"] = redirectUri,
            ["response_type"] = "code",
            ["scope"] = scope,
            ["access_type"] = "offline", // Important: get refresh token
            ["prompt"] = "consent"       // Force consent to get refresh token
        };

        var query = string.Join("&", parameters.Select(
            kvp => $"{Uri.EscapeDataString(kvp.Key)}={Uri.EscapeDataString(kvp.Value)}"));

        return $"{AuthEndpoint}?{query}";
    }

    public async Task<TokenResponse> ExchangeCodeForTokensAsync(
        string code,
        string redirectUri,
        CancellationToken ct = default)
    {
        _logger.LogInformation("Exchanging authorization code for tokens");

        var content = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["client_id"] = _clientId,
            ["client_secret"] = _clientSecret,
            ["code"] = code,
            ["redirect_uri"] = redirectUri,
            ["grant_type"] = "authorization_code"
        });

        var response = await _httpClient.PostAsync(TokenEndpoint, content, ct);
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync(ct);
        var tokenResponse = JsonSerializer.Deserialize<TokenResponse>(json)
            ?? throw new InvalidOperationException("Failed to deserialize token response");

        _logger.LogInformation("Successfully obtained tokens");

        return tokenResponse;
    }

    public async Task<TokenResponse> RefreshAccessTokenAsync(
        string refreshToken,
        CancellationToken ct = default)
    {
        _logger.LogDebug("Refreshing access token");

        var content = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["client_id"] = _clientId,
            ["client_secret"] = _clientSecret,
            ["refresh_token"] = refreshToken,
            ["grant_type"] = "refresh_token"
        });

        var response = await _httpClient.PostAsync(TokenEndpoint, content, ct);
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync(ct);
        var tokenResponse = JsonSerializer.Deserialize<TokenResponse>(json)
            ?? throw new InvalidOperationException("Failed to deserialize token response");

        _logger.LogDebug("Successfully refreshed access token");

        return tokenResponse;
    }
}
```

### OAuth2 Authenticator

```csharp
public class OAuth2Authenticator : IAuthenticator
{
    private readonly IOAuth2Provider _provider;
    private readonly IConfigManager _configManager;
    private readonly ILogger<OAuth2Authenticator> _logger;

    public OAuth2Authenticator(
        IOAuth2Provider provider,
        IConfigManager configManager,
        ILogger<OAuth2Authenticator> logger)
    {
        _provider = provider;
        _configManager = configManager;
        _logger = logger;
    }

    public async Task AuthenticateAsync(
        IImapClient client,
        AccountConfig account,
        CancellationToken ct = default)
    {
        // Check if we have a valid access token
        if (!string.IsNullOrWhiteSpace(account.AccessToken) &&
            account.TokenExpiry.HasValue &&
            DateTime.UtcNow < account.TokenExpiry.Value)
        {
            _logger.LogDebug(
                "Using cached access token for {Account}",
                account.Name);

            await AuthenticateWithTokenAsync(
                client,
                account.Username,
                account.AccessToken,
                ct);
            return;
        }

        // Need to refresh token
        if (string.IsNullOrWhiteSpace(account.RefreshToken))
            throw new InvalidOperationException(
                $"No refresh token available for account '{account.Name}'. " +
                "Run setup command first.");

        _logger.LogInformation(
            "Refreshing access token for {Account}",
            account.Name);

        var tokenResponse = await _provider.RefreshAccessTokenAsync(
            account.RefreshToken,
            ct);

        // Update account with new token
        account.AccessToken = tokenResponse.AccessToken;
        account.TokenExpiry = DateTime.UtcNow
            .AddSeconds(tokenResponse.ExpiresIn)
            .AddMinutes(-5); // 5-minute buffer

        // Save updated configuration
        await _configManager.SaveAccountAsync(account, ct);

        // Authenticate with new token
        await AuthenticateWithTokenAsync(
            client,
            account.Username,
            account.AccessToken,
            ct);

        _logger.LogInformation(
            "Successfully authenticated {Account} with refreshed token",
            account.Name);
    }

    private async Task AuthenticateWithTokenAsync(
        IImapClient client,
        string username,
        string accessToken,
        CancellationToken ct)
    {
        var oauth2 = new SaslMechanismOAuth2(username, accessToken);
        await client.AuthenticateAsync(oauth2, ct);
    }
}
```

### OAuth2 Setup Command

Interactive command for initial OAuth2 setup:

```csharp
public class SetupOAuth2Command
{
    private readonly IOAuth2Provider _provider;
    private readonly IConfigManager _configManager;
    private readonly ILogger<SetupOAuth2Command> _logger;

    private const string RedirectUri = "http://localhost:8080";
    private const int RedirectPort = 8080;

    public async Task ExecuteAsync(
        string accountName,
        CancellationToken ct = default)
    {
        _logger.LogInformation(
            "Starting OAuth2 setup for account '{Account}'",
            accountName);

        // Get authorization URL
        var authUrl = _provider.GetAuthorizationUrl(
            RedirectUri,
            Array.Empty<string>()); // Use default scopes

        _logger.LogInformation("Opening browser for authorization...");
        _logger.LogInformation("URL: {Url}", authUrl);

        // Open browser
        OpenBrowser(authUrl);

        // Start local HTTP server to receive callback
        var authCode = await ListenForCallbackAsync(ct);

        if (string.IsNullOrWhiteSpace(authCode))
            throw new InvalidOperationException("Failed to receive authorization code");

        _logger.LogInformation("Authorization code received, exchanging for tokens...");

        // Exchange code for tokens
        var tokenResponse = await _provider.ExchangeCodeForTokensAsync(
            authCode,
            RedirectUri,
            ct);

        if (string.IsNullOrWhiteSpace(tokenResponse.RefreshToken))
            throw new InvalidOperationException(
                "No refresh token received. Ensure 'access_type=offline' is set.");

        _logger.LogInformation("Tokens obtained successfully");

        // Load config and update account
        var config = await _configManager.LoadAsync(ct);
        var account = config.Accounts.FirstOrDefault(a => a.Name == accountName);

        if (account == null)
            throw new InvalidOperationException(
                $"Account '{accountName}' not found in configuration");

        account.RefreshToken = tokenResponse.RefreshToken;
        account.AccessToken = tokenResponse.AccessToken;
        account.TokenExpiry = DateTime.UtcNow
            .AddSeconds(tokenResponse.ExpiresIn)
            .AddMinutes(-5);

        // Save configuration
        await _configManager.SaveAsync(config, ct);

        _logger.LogInformation(
            "OAuth2 setup complete for account '{Account}'",
            accountName);
        _logger.LogInformation(
            "Refresh token saved to configuration (keep it secure!)");
    }

    private async Task<string?> ListenForCallbackAsync(CancellationToken ct)
    {
        using var listener = new HttpListener();
        listener.Prefixes.Add($"{RedirectUri}/");
        listener.Start();

        _logger.LogInformation("Waiting for OAuth callback on {Uri}...", RedirectUri);

        var context = await listener.GetContextAsync().WaitAsync(ct);
        var code = context.Request.QueryString["code"];
        var error = context.Request.QueryString["error"];

        // Send response to browser
        var responseHtml = string.IsNullOrWhiteSpace(error)
            ? "<html><body><h1>Authorization successful!</h1><p>You can close this window.</p></body></html>"
            : $"<html><body><h1>Authorization failed</h1><p>Error: {error}</p></body></html>";

        var buffer = Encoding.UTF8.GetBytes(responseHtml);
        context.Response.ContentLength64 = buffer.Length;
        context.Response.ContentType = "text/html";
        await context.Response.OutputStream.WriteAsync(buffer, ct);
        context.Response.Close();

        listener.Stop();

        if (!string.IsNullOrWhiteSpace(error))
            throw new InvalidOperationException($"OAuth2 error: {error}");

        return code;
    }

    private void OpenBrowser(string url)
    {
        try
        {
            Process.Start(new ProcessStartInfo
            {
                FileName = url,
                UseShellExecute = true
            });
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "Failed to open browser automatically");
            _logger.LogInformation("Please open this URL manually: {Url}", url);
        }
    }
}
```

## Authenticator Factory

```csharp
public interface IAuthenticatorFactory
{
    IAuthenticator CreateAuthenticator(AccountConfig account);
}

public class AuthenticatorFactory : IAuthenticatorFactory
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<AuthenticatorFactory> _logger;

    public AuthenticatorFactory(
        IServiceProvider serviceProvider,
        ILogger<AuthenticatorFactory> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    public IAuthenticator CreateAuthenticator(AccountConfig account)
    {
        return account.AuthType.ToLowerInvariant() switch
        {
            "password" => _serviceProvider.GetRequiredService<PasswordAuthenticator>(),
            "oauth2" => CreateOAuth2Authenticator(account),
            _ => throw new NotSupportedException(
                $"Authentication type '{account.AuthType}' is not supported")
        };
    }

    private IAuthenticator CreateOAuth2Authenticator(AccountConfig account)
    {
        var provider = DetermineOAuth2Provider(account);

        return new OAuth2Authenticator(
            provider,
            _serviceProvider.GetRequiredService<IConfigManager>(),
            _serviceProvider.GetRequiredService<ILogger<OAuth2Authenticator>>());
    }

    private IOAuth2Provider DetermineOAuth2Provider(AccountConfig account)
    {
        // Determine provider based on server hostname
        var server = account.Server.ToLowerInvariant();

        if (server.Contains("gmail") || server.Contains("google"))
        {
            return new GmailOAuth2Provider(
                account.ClientId ?? throw new InvalidOperationException("ClientId required"),
                account.ClientSecret ?? throw new InvalidOperationException("ClientSecret required"),
                _serviceProvider.GetRequiredService<ILogger<GmailOAuth2Provider>>());
        }

        if (server.Contains("outlook") || server.Contains("office365"))
        {
            return new OutlookOAuth2Provider(
                account.ClientId ?? throw new InvalidOperationException("ClientId required"),
                account.ClientSecret ?? throw new InvalidOperationException("ClientSecret required"),
                _serviceProvider.GetRequiredService<ILogger<OutlookOAuth2Provider>>());
        }

        throw new NotSupportedException(
            $"OAuth2 provider not recognized for server '{account.Server}'");
    }
}
```

## Dependency Injection Setup

```csharp
public static class AuthenticationServiceExtensions
{
    public static IServiceCollection AddAuthentication(this IServiceCollection services)
    {
        // Authenticators
        services.AddScoped<PasswordAuthenticator>();
        services.AddScoped<IAuthenticatorFactory, AuthenticatorFactory>();

        // Commands
        services.AddScoped<SetupOAuth2Command>();

        return services;
    }
}
```

## Usage Examples

### Command Line Interface

```bash
# Setup Gmail OAuth2
dotnet run -- setup-oauth gmail work

# Setup with custom redirect port
dotnet run -- setup-oauth gmail work --port 9090

# Test authentication
dotnet run -- test-auth work

# Run with authenticated account
dotnet run
```

### Programmatic Usage

```csharp
// Create authenticator
var factory = serviceProvider.GetRequiredService<IAuthenticatorFactory>();
var authenticator = factory.CreateAuthenticator(account);

// Connect and authenticate
using var client = new ImapClientWrapper();
await client.ConnectAsync(account.Server, account.Port, account.UseSsl, ct);
await authenticator.AuthenticateAsync(client, account, ct);

// Use client...
```

## Security Best Practices

### 1. Token Storage

**DO**:
- Encrypt refresh tokens before storing
- Store tokens in user-specific locations (e.g., `~/.config/imap-processor/`)
- Use OS-level encryption (DPAPI on Windows, Keychain on macOS)
- Add token files to `.gitignore`

**DON'T**:
- Store refresh tokens in plain text
- Commit tokens to version control
- Share token files between users

### 2. Token Refresh

- Refresh tokens 5 minutes before expiry
- Handle refresh failures gracefully (prompt for re-authentication)
- Rate-limit refresh attempts to prevent abuse

### 3. Error Handling

```csharp
try
{
    await authenticator.AuthenticateAsync(client, account, ct);
}
catch (AuthenticationException ex)
{
    _logger.LogError(ex, "Authentication failed for {Account}", account.Name);

    if (account.AuthType == "oauth2")
    {
        _logger.LogWarning(
            "OAuth2 tokens may be expired. Run setup command to re-authenticate: " +
            "dotnet run -- setup-oauth {Provider} {Account}",
            DetermineProvider(account),
            account.Name);
    }

    throw;
}
```

## Testing OAuth2 Flow

### Manual Testing Steps

1. **Setup account**:
   ```bash
   dotnet run -- setup-oauth gmail work
   ```

2. **Verify tokens saved**:
   Check `appsettings.json` for `refreshToken` field

3. **Test authentication**:
   ```bash
   dotnet run -- test-auth work
   ```

4. **Test token refresh**:
   - Delete `accessToken` from config
   - Run authentication again
   - Verify new token is obtained

### Unit Testing

```csharp
public class OAuth2AuthenticatorTests
{
    [Fact]
    public async Task AuthenticateAsync_WithValidCachedToken_UsesCache()
    {
        // Arrange
        var provider = new Mock<IOAuth2Provider>();
        var configManager = new Mock<IConfigManager>();
        var logger = new Mock<ILogger<OAuth2Authenticator>>();

        var account = new AccountConfig
        {
            Name = "test",
            Username = "user@example.com",
            AccessToken = "valid_token",
            TokenExpiry = DateTime.UtcNow.AddHours(1),
            RefreshToken = "refresh_token"
        };

        var client = new Mock<IImapClient>();

        var authenticator = new OAuth2Authenticator(
            provider.Object,
            configManager.Object,
            logger.Object);

        // Act
        await authenticator.AuthenticateAsync(client.Object, account);

        // Assert
        client.Verify(c => c.AuthenticateAsync(
            It.IsAny<SaslMechanism>(),
            It.IsAny<CancellationToken>()), Times.Once);

        provider.Verify(p => p.RefreshAccessTokenAsync(
            It.IsAny<string>(),
            It.IsAny<CancellationToken>()), Times.Never);
    }
}
```

## Extending to Other Providers

### Outlook/Microsoft 365

```csharp
public class OutlookOAuth2Provider : IOAuth2Provider
{
    private const string AuthEndpoint = "https://login.microsoftonline.com/common/oauth2/v2.0/authorize";
    private const string TokenEndpoint = "https://login.microsoftonline.com/common/oauth2/v2.0/token";
    private const string Scope = "https://outlook.office365.com/IMAP.AccessAsUser.All offline_access";

    public string ProviderName => "Microsoft";

    // Implementation similar to GmailOAuth2Provider...
}
```

### Yahoo

```csharp
public class YahooOAuth2Provider : IOAuth2Provider
{
    private const string AuthEndpoint = "https://api.login.yahoo.com/oauth2/request_auth";
    private const string TokenEndpoint = "https://api.login.yahoo.com/oauth2/get_token";
    private const string Scope = "mail-r mail-w";

    public string ProviderName => "Yahoo";

    // Implementation similar to GmailOAuth2Provider...
}
```

## Troubleshooting

### Common Issues

**Issue**: "No refresh token received"
- **Cause**: Missing `access_type=offline` or `prompt=consent`
- **Fix**: Ensure OAuth URL includes both parameters

**Issue**: "Token expired"
- **Cause**: Access token expired and refresh failed
- **Fix**: Re-run setup command to get new tokens

**Issue**: "Invalid grant"
- **Cause**: Refresh token was revoked or expired
- **Fix**: Re-run setup command to re-authorize

**Issue**: "Redirect URI mismatch"
- **Cause**: Redirect URI doesn't match Google Cloud Console configuration
- **Fix**: Add `http://localhost:8080` to authorized redirect URIs
