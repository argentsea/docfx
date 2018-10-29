# Security

ArgentSea encourages better [security practices](https://www.owasp.org/index.php/.NET_Security_Cheat_Sheet) in two ways:

* By making it easier to store password information securely.
* By enforcing the use of stored procedures / functions.

## Secure Password Storage

ArgentSea’s data connection metadata divorces passwords from a legacy “connection string” implementation. Because the new .NET configuration architecture enables configuration data to be distributed among multiple sources, passwords can be stored in secure locations — independently of the rest of the connection metadata.

For example, passwords can stored in:
* [UserSecrets](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets) — in a development context.
* [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
* [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/)
* Secure, encrypted file shares

Each provider has a NuGet package which simplifies loading their secrets into your application.

If your password store handles JSON, you can add credentials like this:

```json
"Credentials": [
  {
    "SecurityKey": "SecKey1",
    "UserName": "webuser",
    "Password": "D~|Y98N?A9qkEz^u%Dhn"
  },
  {
    "SecurityKey": "SecKey2",
    "UserName": "adminUser",
    "Password": "A*dNppRZq3-vE@Ci"
  }
]
```

If your password store uses key-value pairs, you can store the same information like this:

| Key | Value |
| --- | --- |
| Credentials:0:SecurityKey | SecKey1 |
| Credentials:0:UserName | SecKey1 |
| Credentials:0:Password | SecKey1 |
| Credentials:1:SecurityKey | SecKey2 |
| Credentials:1:UserName | adminUser |
| Credentials:1:Password | A*dNppRZq3-vE@Ci |

Connections that use Windows authentication do not need to be stored securely. These entries can be added directly to *appsettings.json*:

```json
  "Credentials": [
    {
      "SecurityKey": "SecKey1",
      "WindowsAuth": true
    }
  ],
```

## Stored procedures/functions

ArgentSea enforces the use of stored procedures or functions. Besides offering performance advantages, this provides substantial security benefits.

### Least Privileged Access

When data access is managed by stored procedures or functions exclusively, permissions can be granted exclusively to the procedures and not to the underlying tables. If an attacker were to take ownership of the connection, they would not be able to perform any database activity that was not explicitly enabled by the existing procedures or functions.

### Injection Attacks

A common data hack involves crafting the requests sent to a database server to include additional commands. Stored procedures/functions with parameters are immune to this attack, assuming no dynamic is used SQL within the procedure body.

### DBA Validation

When dynamic SQL commands are created by the application, it can be difficult to debug the command sequence, even harder to correct it, and nearly impossible to constrain it to appropriate activity. When using stored procedures or functions, however, DBAs can review the implementation of dangerous commands, and ensure that their explicit use is appropriate.