+++
title = "technical breakdown: vault"
date = 2024-11-10
+++

I released my credential manager last month and am finally getting around to documenting it. This project represents a complete redesign of my previous work, featuring an authentication system and web-based architecture. I've maintained the use of AES symmetric encryption from my previous solution as it remains ideal for securely storing credentials.

**From Desktop to Web:**

My original credential manager was built as a terminal application in C# with .NET 7. While functional and cross-platform, it required executing an `.exe` file and had limited portability compared to web-based solutions. The core functionality was straightforward: users would create an AES key and credential store, then upload both to modify their stored credentials.

The [new implementation](https://github.com/chomnr/vault) uses a completely different technology stack with Next.js and TypeScript for the frontend, Next.js API routes for the backend, MongoDB with Prisma ORM for data storage, and AES encryption with iron-session for authentication. This technology selection was deliberate, creating a more accessible and maintainable solution while preserving strong security principles.

**Database Architecture:**

Rather than storing credentials in files, I chose MongoDB as a NoSQL database solution. This decision was driven by the need for flexibility rather than rigid structure—perfect for a credential manager where different credential types might require different data fields.

Here's the Prisma schema defining the data models:

```typescript
model Vault {
  id              String       @id @default(uuid()) @map("_id")
  name            String
  maxCredentials  Int
  credentials     Credential[]
  secret          String
  iv              String
}

model Credential {
  id          String   @id @default(uuid()) @map("_id")
  type        String
  name        String
  iv          String
  data        String
  createdAt   DateTime @default(now())
  updatedAt   DateTime

  vaultId     String
  vault       Vault    @relation(fields: [vaultId], references: [id])
}
```

Each vault contains a unique identifier, name and capacity limit, encrypted validation string (the "secret"), initialization vector (IV) for encryption, and relationship to multiple credentials. The encryption strategy is selective—only the `data` field in the `Credential` model is encrypted. This approach maintains security for sensitive information while keeping metadata like credential type, name, and timestamps accessible for filtering and organization without requiring decryption.

The `secret` and `iv` fields in the `Vault` model serve a validation purpose. Rather than storing the actual encryption key, I encrypt a known string ("secret") and store the result. When a user provides their key, the system attempts to decrypt this validation string—if successful, the key is proven valid.

**Authentication System:**

The application follows a "one user, multiple vaults" design philosophy. I intentionally hardcoded credentials in environment variables to prevent multiple user accounts, encouraging shared access with separate vaults for different purposes or users.

Session management is handled through the iron-session library, which provides secure cookie-based authentication:

```typescript
export async function isAuthenticated() {
  const session = await getIronSession(cookies(), SESSION_OPTIONS)
  const COOKIE_AGE_OFFSET = COOKIE_MAX_AGE(session?.remember) * 1000

  if (
    !session ||
    Object.keys(session).length === 0 ||
    Date.now() > session.timeStamp + COOKIE_AGE_OFFSET
  ) {
    return false
  } else {
    return true
  }
}
```

This implementation retrieves the encrypted cookie from the request, verifies the session's validity and expiration, and works alongside middleware to protect authenticated routes. This approach provides a defense-in-depth strategy—even if the authentication is compromised, attackers still need the correct AES key to access any vault data.

**Encryption Implementation:**

I maintained AES as the encryption algorithm, using AES-CBC with 256-bit keys for industry-standard security:

```typescript
const generatedKey = randomBytes(32) // 256 bits
const cipher = forge.cipher.createCipher(
  'AES-CBC',
  forge.util.createBuffer(generatedKey)
)
cipher.start({iv})
cipher.update(forge.util.createBuffer('secret'))
const success = cipher.finish()

if (!success)
  return err_route(
    VAULT_ENCRYPTION_FAILED.status,
    VAULT_ENCRYPTION_FAILED.msg,
    VAULT_ENCRYPTION_FAILED.code
  )

const encrypted = cipher.output
const headers = new Headers()
headers.set('Content-Disposition', 'attachment; filename="aes-key.aes"')
headers.set('Content-Type', 'application/octet-stream')

await prisma.vault.create({
  data: {
    name: name,
    maxCredentials: maxCredentials,
    secret: Buffer.from(encrypted.getBytes(), 'binary').toString('base64'),
    iv: Buffer.from(iv, 'binary').toString('base64')
  }
})

prisma.$disconnect()
return new NextResponse(Buffer.from(generatedKey).toString('base64'), {
  status: 200,
  headers
})
```

When a user creates a vault, the system generates a cryptographically secure 256-bit key, encrypts a known validation string ("secret") with this key, stores the encrypted validation string and IV in the database, returns the actual encryption key to the user for safekeeping, and requires this key for future access to the vault. This approach ensures the encryption key never remains on the server, maintaining a zero-knowledge security model.

**Key Improvements:**

This credential manager represents a major improvement over its predecessor, transitioning from a terminal-based application to a modern web solution while preserving strong security principles. The web-based interface is accessible from any modern browser, the NoSQL database accommodates various credential types, improved security maintains strong encryption while adding an authentication layer, the user experience is more intuitive than the terminal application, and selective encryption only encrypts sensitive data, improving performance and usability.

The "one user, multiple vaults" approach provides flexibility while maintaining simplicity in the authentication system. Each vault requires its own encryption key, ensuring compartmentalized security even when multiple users share the same login. While there are still opportunities for improvement (such as refining the session expiration mechanism), the current implementation successfully balances security, usability, and modern web architecture.