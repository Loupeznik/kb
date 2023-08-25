# Implementing custom identity provider
## Basic JWT example (.NET Core)
1. Install `Microsoft.AspNetCore.Authentication.JwtBearer` NuGet
2. Add *appsettings* variables (issuer, audience, base64-encoded JWK, default expiration, etc.)
3. Add configuration (to *Program.cs* or custom configuration extensions)
```csharp
// ...

var jwtConfig = builder.Configuration.GetSection("Identity").Get<IdentityConfiguration>();

// ...

builder.Services
    .AddAuthentication()
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtConfig.Issuer,
            ValidAudiences = jwtConfig.Audience,
            IssuerSigningKey = new JsonWebKey(Encoding.ASCII.GetString(Convert.FromBase64String(jwtConfig.Key)))
        };
    });
```
4. Generate JWK ([mkjwk.org](https://mkjwk.org/))
	1. Example parameters:
		1. Key size: 2048
		2. Key use: Signature
		3. Algorithm: RS256
		4. KeyID: SHA256
	2. After the key is generated, copy and store the keypair
5. Add encoded JWK to secrets
6. Create a token issuer
```csharp
//...
    public string GenerateToken() => new JwtSecurityTokenHandler()
            .WriteToken(new JwtSecurityToken(_config.Issuer,
                _config.Audience.FirstOrDefault(),
                claims: new List<Claim>
                {
                    new Claim(ClaimTypes.Name, "test"),
                    new Claim(ClaimTypes.Role, ApiRole.App)
                },
                expires: DateTime.Now.AddHours(12),
                signingCredentials: new SigningCredentials(new JsonWebKey(Encoding.ASCII.GetString(Convert.FromBase64String(_config.Key!))), SecurityAlgorithms.RsaSha256)));
//...
```
7. Implement endpoints for obtaining the token, add authorization to the API
