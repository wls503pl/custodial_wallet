# Wallet Signer Design & Account Generation

## Part 1: Digital Wallet Fundamentals

### What is a Digital Wallet?

In blockchain, a wallet isn't actually a place to store money—it's a **tool to manage private keys**. Your digital assets live on the blockchain. Only by possessing the private key can you control these assets.

```
Private Key → Elliptic Curve Algorithm → Public Key → Hash Function → Address
  (Secret)          (One-way)                (Public)
```

**Core Principle:** Private key generates public key, public key generates address. Both steps are one-way and irreversible.

### Step One: Generating the Private Key

A private key is essentially a random number selected between 1 and 2^256. The crucial part is that this number must be sufficiently **random** and **unpredictable**.

**Analogy:** It's like flipping a coin 256 times. Each heads = 1, each tails = 0. The resulting 256-bit binary number is your private key.

---

## Part 2: HD Wallets & Hierarchical Derivation (BIP32)

**BIP32** enables one seed to derive unlimited private keys through a deterministic algorithm.

![Hierarchical Derivation: Seed (pentagon) → Master Private Key (km) → Multiple child keys (km/0, km/1, km/2, km/3) → Each generates an address](../../img/cn/02_signer_accountGeneration/account_generation.png)

---

## Part 3: Path Standards & Multi-Coin Support (BIP44)

### Path Notation

Derived keys are represented using tree-like paths, separated by slashes `/`:

```
m/0/1/2  (m is the master private key, followed by derivation levels)
M/0/1/2  (M is the master public key)
```

### BIP44 Standard Path

**Problem:** Different wallets used different path standards, causing confusion.

**Solution:** BIP44 defines a unified 5-layer tree structure:

```
m / purpose' / coin' / account' / change / address_index
```

**Component Meanings:**

| Component     | Meaning            | Example                |
| ------------- | ------------------ | ---------------------- |
| m             | Master private key | Fixed                  |
| purpose'      | Purpose            | Fixed as 44'           |
| coin'         | Coin type          | 0=Bitcoin, 60=Ethereum |
| account'      | Account index      | 0, 1, 2...             |
| change        | Address type       | 0=receive, 1=change    |
| address_index | Address sequence   | 0, 1, 2...             |

### Ethereum Real-World Path

```
m/44'/60'/0'/0/0
   ↓  ↓  ↓ ↓ ↓
 fixed Ethereum account0 receive 0th address

m/44'/60'/0'/0/1
   ↓  ↓  ↓ ↓ ↓
 fixed Ethereum account0 receive 1st address
```

---

## Part 4: Seed Phrase Backup (BIP39)

### The Problem: How to Safely Back Up the Seed?

A random seed is a long hexadecimal string, difficult to memorize or back up:

```
090ABCB3A6e1400e9345bC60c78a8BE7  ← This is too cumbersome to backup
```

### The Solution: 12 Words Instead

**BIP39** converts the random seed into 12 or 24 common English words:

```
candy maple cake sugar pudding cream honey rich
smooth crumble sweet treat
```

**Conversion Process:**

1. Generate 128-bit random number
2. Calculate checksum (4 bits)
3. Split into 12 × 11-bit binary numbers
4. Map each number to a word from the BIP39 wordlist

### Recovering Accounts from Seed Phrase

Seed Phrase + Password → PBKDF2 Computation → Seed → Derive Private Keys → Addresses

**Key Point:** Same seed phrase + password always generates the same accounts. Different passwords generate different accounts.

---

## Part 5: Exchange Wallet System Architecture

![Exchange Wallet System Architecture: Wallet module receives user deposit requests and broadcasts. Scan monitors blockchain deposits. Risk Control validates transactions. Multiple Signer machines (internal network isolated) sign transactions. Fund Rebalance consolidates to cold wallets. Internal communication via dotted lines, blockchain communication via solid lines.](../../img/cn/02_signer_accountGeneration/modular_walletSystem.png)

### Database Design

#### Wallet Main Module Database

**Users Table:** Records user basic info (username, email, password, KYC status, etc.)

**Wallets Table:** Records user address information

| Field      | Description                             |
| ---------- | --------------------------------------- |
| user_id    | User ID                                 |
| address    | User wallet address                     |
| device     | Which Signer device generated it        |
| path       | Derivation path (e.g. m/44'/60'/0'/0/0) |
| chain_type | Coin type (evm, btc, solana)            |

#### Signer Database

| Field       | Description             |
| ----------- | ----------------------- |
| address     | Generated address       |
| path        | Derivation path         |
| index_value | Index value in the path |
| chain_type  | Coin type               |

**Security Note:** Neither database stores the private key, only the derivation path. Private keys never touch disk.

---

## Part 6: Signer Security Design

### Password Protection

When starting the signer, operators must enter a password:

```
Environment Variable: Seed Phrase (stored in server config)
+
Operator Input: Password (hidden input, displayed as *)
        ↓
Generate Seed → Derive Private Keys → Generate Addresses
```

**Security Improvements:**

- Even if attacker obtains the seed phrase, they can't generate private keys without the password
- Hidden input prevents password from being recorded in terminal history
- Verify first address on startup to ensure password is correct

### Address Generation Flow

```
POST /api/signer/create?chainType=evm
    ↓
Generate next path based on current max index
    ↓
m/44'/60'/0'/0/{nextIndex}
    ↓
Seed Phrase + Password → Generate Seed
    ↓
Derive private key from seed via path
    ↓
Private key → Generate public key and address
    ↓
Save address, path, index to database
    ↓
Return address (only address, not private key)
```

---

## Part 7: Complete Flow from Seed to Address

```
1. Generate Seed Phrase (BIP39)
   ↓
   candy maple cake sugar ... (12 words)

2. Seed Phrase + Password → Seed (Key Stretching)
   ↓
   A 512-bit random number

3. Seed → Master Private/Public Key (BIP32)
   ↓
   HMAC-SHA512 computation

4. Hierarchical Derivation (BIP32)
   ↓
   m/44'/60'/0'/0/0 → First account
   m/44'/60'/0'/0/1 → Second account
   ...

5. Private Key → Public Key → Address (Elliptic Curve + Hash)
   ↓
   Final blockchain address ready to receive transfers
```

---

## Part 8: Why This Design?

| Design                         | Problem Solved                                                                                              |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| **HD Wallet (BIP32/BIP39)**    | No need to backup many private keys; secure one seed phrase                                                 |
| **Standard Path (BIP44)**      | Unified standard; different wallets can recover the same accounts                                           |
| **Wallet-Signer Separation**   | Private keys isolated from business logic; prevents private key theft even if business layer is compromised |
| **Internal Network Isolation** | Signer only accepts internal requests; prevents external attacks                                            |
| **Seed Phrase + Password**     | Dual-factor protection; enhanced security                                                                   |

---

## Part 9: Implementation Details

### Tech Stack

- **Runtime:** Node.js + Express
- **Database:** SQLite3 (minimal external dependencies)
- **Libraries:**
  - `@scure/bip39`: Seed phrase generation and validation
  - `@scure/bip32`: HD key derivation
  - `viem`: Ethereum account creation

### Key Implementation

#### Password Protection on Startup

```typescript
// Load seed phrase from environment
const mnemonic = process.env.MNEMONIC;

// Interactive password input (displays as *)
const password = await readHiddenInput("Enter signer password: ");

// Verify with first address
const seed = mnemonicToSeedSync(mnemonic, password);
const firstAddress = deriveAddress(seed, "m/44'/60'/0'/0/0");

if (firstAddress !== storedFirstAddress) {
  console.error("Password incorrect!");
  process.exit(1);
}
```

#### Create New Address

```typescript
async createNewWallet(chainType: 'evm' | 'btc' | 'solana') {
  // Get seed phrase from environment
  const mnemonic = this.getMnemonicFromEnv();

  // Generate next derivation path
  const derivationPath = await this.generateNextPath(chainType);

  // Create account from private key
  const seed = mnemonicToSeedSync(mnemonic, this.password);
  const hdKey = HDKey.fromMasterSeed(seed);
  const derivedKey = hdKey.derive(derivationPath);

  // Get address (private key never exposed)
  const privateKeyHex = `0x${Buffer.from(derivedKey.privateKey).toString('hex')}`;
  const account = privateKeyToAccount(privateKeyHex);

  // Save to database (only address and path, not private key)
  await this.db.saveAddress(account.address, derivationPath);

  return account.address;
}
```

#### Get Next Derivation Path

```typescript
async generateNextPath(chainType: 'evm'): Promise<string> {
  // Base path: m/44'/60'/0'/0
  const basePath = "m/44'/60'/0'/0";

  // Get current max index from database
  const maxIndex = await this.db.getMaxIndexForChain(chainType);

  // Next index = maxIndex + 1
  const nextIndex = maxIndex + 1;

  return `${basePath}/${nextIndex}`;
}
```

### Address Generation Performance

For high-concurrency scenarios, exchanges use a hybrid approach:

**Batch Pre-generation:** For popular chains, pre-generate a pool of addresses. When the pool runs low, request more from Signer.

**On-demand Generation:** For less popular chains, generate addresses in real-time.

**Shared Address:** Some exchanges have multiple users share one deposit address and differentiate by memo/tag field.

---

## Part 10: FAQ

**Q: What if I lose my seed phrase?**
A: It cannot be recovered. This is why offline cold backup is essential. Store in a physically secure location (like a safe).

**Q: Can one person have multiple accounts?**
A: Yes. Change the `account'` or `address_index` to generate different accounts. All controlled by one seed phrase.

**Q: Can Bitcoin and Ethereum addresses be mixed?**
A: No. Different `coin_type` values (Bitcoin=0, Ethereum=60) derive different private keys. The address formats are also incompatible.

**Q: How many seed phrases should an exchange back up?**
A: One is enough technically. However, typically multiple cold backups are maintained in separate locations to prevent single points of failure.

**Q: Why use a password in addition to the seed phrase?**
A: Adds a second layer of security. Even if someone steals the seed phrase, they still need the password to derive private keys.

**Q: What if a Signer server is compromised?**
A: Private keys are never stored on disk and never leave the Signer process. The attacker would need both the seed phrase AND the password, which are stored separately.

---

## Summary

The exchange wallet system implements a secure, scalable architecture for managing user accounts:

1. **Security**: Private keys isolated in Signer module, internal network protected
2. **Scalability**: Single seed phrase generates unlimited addresses via BIP32/BIP44
3. **Recoverability**: Seed phrase provides full account recovery capability
4. **Maintainability**: Clear separation of concerns between Wallet and Signer modules
5. **Compliance**: Deterministic derivation allows auditing and reconstruction of account history

This design balances security, performance, and operational efficiency—essential for a production exchange system.
