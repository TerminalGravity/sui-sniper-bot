# sui-sniper-bot

Certainly! Below is the **finalized TypeScript** code for your Sui blockchain sniper bot, incorporating the following configurations:

- **Slippage**: Set to **25%**.
- **Minimum Liquidity Amount**: Set to **10,000** (10K).
- **Buy Amount**: Set to **50 SUI**.

Additionally, I have provided a comprehensive **step-by-step guide** on how to set up, configure, and launch the bot.

---

## **Final Sniper Bot Code (`sniperBot.ts`)**

```typescript
import { 
    Connection, 
    JsonRpcProvider, 
    RawSigner, 
    Ed25519Keypair 
} from '@mysten/sui.js';
import axios from 'axios';
import * as dotenv from 'dotenv';
dotenv.config();

// --------------------- Configuration ---------------------

// Environment Variables
const PRIVATE_KEY = process.env.PRIVATE_KEY || '';
const RPC_URL = process.env.RPC_URL || 'https://fullnode.devnet.sui.io';
const INSIDE_API_URL = process.env.INSIDE_API_URL || 'https://api.inside.io/getContractScore';
const INSIDE_API_KEY = process.env.INSIDE_API_KEY || '';
const MOVEPUMP_API_URL = process.env.MOVEPUMP_API_URL || 'https://api.movepump.dex/getNewContracts';
const MOVEPUMP_API_KEY = process.env.MOVEPUMP_API_KEY || '';
const TWEETSCOUT_API_URL = process.env.TWEETSCOUT_API_URL || 'https://api.tweetscout.io/getTwitterScore';
const TWEETSCOUT_API_KEY = process.env.TWEETSCOUT_API_KEY || '';
const SCORE_THRESHOLD = parseInt(process.env.SCORE_THRESHOLD || '80', 10);
const TWITTER_SCORE_THRESHOLD = parseInt(process.env.TWITTER_SCORE_THRESHOLD || '50', 10);

// Trading Configurations
const SLIPPAGE = parseFloat(process.env.SLIPPAGE || '0.25'); // 25%
const MIN_LIQUIDITY = parseInt(process.env.MIN_LIQUIDITY || '10000', 10); // 10K
const BUY_AMOUNT_SUI = parseFloat(process.env.BUY_AMOUNT_SUI || '50'); // 50 SUI

// Validate Environment Variables
if (!PRIVATE_KEY) {
    throw new Error('PRIVATE_KEY is not set in the environment variables.');
}
if (!INSIDE_API_KEY) {
    throw new Error('INSIDE_API_KEY is not set in the environment variables.');
}
if (!MOVEPUMP_API_KEY) {
    throw new Error('MOVEPUMP_API_KEY is not set in the environment variables.');
}
if (!TWEETSCOUT_API_KEY) {
    throw new Error('TWEETSCOUT_API_KEY is not set in the environment variables.');
}

// DEX Endpoints and Configurations
const DEXs = {
    BlueMove: {
        name: 'BlueMove',
        endpoint: 'https://api.bluemove.dex/executeTrade', // Replace with actual BlueMove DEX endpoint
        apiKey: process.env.BLUEMOVE_API_KEY || '', // Replace if BlueMove requires an API key
    },
    Cetus: {
        name: 'Cetus',
        endpoint: 'https://api.cetus.dex/executeTrade', // Replace with actual Cetus DEX endpoint
        apiKey: process.env.CETUS_API_KEY || '', // Replace if Cetus requires an API key
    },
    MovePump: {
        name: 'MovePump',
        endpoint: 'https://api.movepump.dex/executeTrade', // Replace with actual MovePump DEX endpoint
        apiKey: MOVEPUMP_API_KEY, // Assuming MovePump requires an API key
    },
};

// --------------------- Initialization ---------------------

// Initialize Sui Connection and Signer
const connection = new Connection({ fullnode: RPC_URL });
const provider = new JsonRpcProvider(connection);

// Initialize Keypair from Private Key
let keypair: Ed25519Keypair;
try {
    keypair = Ed25519Keypair.fromSecretKey(Buffer.from(PRIVATE_KEY, 'hex'));
} catch (error) {
    throw new Error('Invalid PRIVATE_KEY format. Ensure it is a hex string without 0x prefix.');
}

const signer = new RawSigner(keypair, provider);

// --------------------- Utility Functions ---------------------

/**
 * Fetches new contract addresses from MovePump.
 * Assumes MovePump provides an API endpoint that returns an array of contract addresses.
 */
async function getNewContractsFromMovePump(): Promise<string[]> {
    try {
        const response = await axios.get(MOVEPUMP_API_URL, {
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${MOVEPUMP_API_KEY}`, // Assuming Bearer Token Authentication
            },
            timeout: 10000, // 10 seconds timeout
        });

        if (response.status === 200 && Array.isArray(response.data.contracts)) {
            console.log(`Fetched ${response.data.contracts.length} new contracts from MovePump.`);
            return response.data.contracts;
        } else {
            console.error(`Unexpected response structure from MovePump API:`, response.data);
            return [];
        }
    } catch (error) {
        console.error(`Error fetching new contracts from MovePump:`, error);
        return [];
    }
}

/**
 * Fetches the contract score from Inside for a given contract address.
 * @param contractAddress - The address of the contract to evaluate.
 * @returns The contract score as a number, or null if failed.
 */
async function getContractScore(contractAddress: string): Promise<number | null> {
    try {
        const response = await axios.get(INSIDE_API_URL, {
            params: { contractAddress },
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${INSIDE_API_KEY}`, // Assuming Bearer Token Authentication
            },
            timeout: 5000, // 5 seconds timeout
        });

        if (response.status === 200 && response.data && typeof response.data.score === 'number') {
            return response.data.score;
        } else {
            console.error(`Unexpected response structure from Inside API for contract ${contractAddress}:`, response.data);
            return null;
        }
    } catch (error) {
        console.error(`Error fetching contract score from Inside for contract ${contractAddress}:`, error);
        return null;
    }
}

/**
 * Fetches the Twitter score from TweetScout for a given contract address.
 * @param contractAddress - The address of the contract to evaluate.
 * @returns The Twitter score as a number, or null if failed.
 */
async function getTwitterScore(contractAddress: string): Promise<number | null> {
    try {
        const response = await axios.get(TWEETSCOUT_API_URL, {
            params: { contractAddress },
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${TWEETSCOUT_API_KEY}`, // Assuming Bearer Token Authentication
            },
            timeout: 5000, // 5 seconds timeout
        });

        if (response.status === 200 && response.data && typeof response.data.twitterScore === 'number') {
            return response.data.twitterScore;
        } else {
            console.error(`Unexpected response structure from TweetScout API for contract ${contractAddress}:`, response.data);
            return null;
        }
    } catch (error) {
        console.error(`Error fetching Twitter score from TweetScout for contract ${contractAddress}:`, error);
        return null;
    }
}

/**
 * Checks if a token has a 100% bonding curve.
 * Assumes the bonding curve information can be fetched via an API or contract call.
 * @param contractAddress - The address of the token contract.
 * @returns True if the bonding curve is 100%, false otherwise.
 */
async function isBondingCurve100(contractAddress: string): Promise<boolean> {
    try {
        // Placeholder: Replace with actual logic to fetch and verify the bonding curve
        // This could involve calling a specific function on the contract or querying an API
        // Example using a hypothetical API endpoint:
        const bondingCurveAPI = `https://api.movepump.dex/getBondingCurve?contract=${contractAddress}`;
        const response = await axios.get(bondingCurveAPI, {
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${MOVEPUMP_API_KEY}`, // If required
            },
            timeout: 5000, // 5 seconds timeout
        });

        if (response.status === 200 && response.data && typeof response.data.percentage === 'number') {
            return response.data.percentage === 100;
        } else {
            console.error(`Unexpected response structure when checking bonding curve for ${contractAddress}:`, response.data);
            return false;
        }
    } catch (error) {
        console.error(`Error checking bonding curve for contract ${contractAddress}:`, error);
        return false;
    }
}

/**
 * Checks if the liquidity of the token meets the minimum requirement.
 * @param contractAddress - The address of the token contract.
 * @returns True if liquidity >= MIN_LIQUIDITY, false otherwise.
 */
async function hasMinimumLiquidity(contractAddress: string): Promise<boolean> {
    try {
        // Placeholder: Replace with actual logic to fetch liquidity
        // This could involve querying the DEX or using an API
        // Example using a hypothetical API endpoint:
        const liquidityAPI = `https://api.movepump.dex/getLiquidity?contract=${contractAddress}`;
        const response = await axios.get(liquidityAPI, {
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${MOVEPUMP_API_KEY}`, // If required
            },
            timeout: 5000, // 5 seconds timeout
        });

        if (response.status === 200 && response.data && typeof response.data.liquidity === 'number') {
            return response.data.liquidity >= MIN_LIQUIDITY;
        } else {
            console.error(`Unexpected response structure when checking liquidity for ${contractAddress}:`, response.data);
            return false;
        }
    } catch (error) {
        console.error(`Error checking liquidity for contract ${contractAddress}:`, error);
        return false;
    }
}

/**
 * Executes a trade on a specified DEX.
 * @param dex - The DEX configuration object.
 * @param contractAddress - The address of the token contract to snipe.
 */
async function executeTrade(dex: { name: string; endpoint: string; apiKey?: string }, contractAddress: string) {
    console.log(`\nAttempting to execute trade on ${dex.name} for contract ${contractAddress}...`);

    try {
        // Prepare transaction data based on DEX requirements
        const txData = {
            contractAddress, // Token contract address
            buyAmountSUI: BUY_AMOUNT_SUI, // Buy amount in SUI
            slippage: SLIPPAGE, // Slippage percentage
            // Additional parameters as required by the DEX
        };

        // Prepare headers, including API key if required
        const headers: Record<string, string> = {
            'Content-Type': 'application/json',
        };
        if (dex.apiKey) {
            headers['Authorization'] = `Bearer ${dex.apiKey}`;
        }

        // Send trade request to the DEX's API
        const response = await axios.post(dex.endpoint, txData, {
            headers,
            timeout: 10000, // 10 seconds timeout
        });

        if (response.status === 200) {
            console.log(`Trade executed successfully on ${dex.name} for contract ${contractAddress}`);
            // Optionally, handle the transaction hash or receipt
            // Example:
            // const txHash = response.data.txHash;
            // console.log(`Transaction Hash: ${txHash}`);
        } else {
            console.error(`Trade failed on ${dex.name} for contract ${contractAddress}: ${response.statusText}`);
        }
    } catch (error) {
        console.error(`Error executing trade on ${dex.name} for contract ${contractAddress}:`, error);
    }
}

/**
 * Processes a single contract: checks contract score, Twitter score, bonding curve, and liquidity before trading.
 * @param contractAddress - The address of the contract to process.
 */
async function processContract(contractAddress: string) {
    console.log(`\n[Process] Evaluating contract: ${contractAddress}`);

    // Step 1: Check contract score via Inside
    const score = await getContractScore(contractAddress);
    if (score === null) {
        console.warn(`Failed to retrieve contract score for ${contractAddress}. Skipping.`);
        return;
    }

    console.log(`Contract score for ${contractAddress}: ${score}`);

    if (score < SCORE_THRESHOLD) {
        console.warn(`Contract score (${score}) is below threshold (${SCORE_THRESHOLD}). Skipping.`);
        return;
    }

    // Step 2: Check Twitter score via TweetScout
    const twitterScore = await getTwitterScore(contractAddress);
    if (twitterScore === null) {
        console.warn(`Failed to retrieve Twitter score for ${contractAddress}. Skipping.`);
        return;
    }

    console.log(`Twitter score for ${contractAddress}: ${twitterScore}`);

    if (twitterScore < TWITTER_SCORE_THRESHOLD) {
        console.warn(`Twitter score (${twitterScore}) is below threshold (${TWITTER_SCORE_THRESHOLD}). Skipping.`);
        return;
    }

    // Step 3: Check bonding curve
    const bondingCurveValid = await isBondingCurve100(contractAddress);
    if (!bondingCurveValid) {
        console.warn(`Bonding curve for ${contractAddress} is not 100%. Skipping.`);
        return;
    }

    console.log(`Bonding curve for ${contractAddress} is 100%.`);

    // Step 4: Check minimum liquidity
    const liquidityValid = await hasMinimumLiquidity(contractAddress);
    if (!liquidityValid) {
        console.warn(`Liquidity for ${contractAddress} is below minimum (${MIN_LIQUIDITY}). Skipping.`);
        return;
    }

    console.log(`Liquidity for ${contractAddress} meets the minimum requirement.`);

    console.log(`All checks passed for ${contractAddress}. Proceeding to snipe.`);

    // Step 5: Execute trade on all supported DEXs
    const dexPromises = Object.values(DEXs).map(dex => executeTrade(dex, contractAddress));
    await Promise.all(dexPromises);
}

// --------------------- Core Bot Functionality ---------------------

/**
 * Monitors MovePump for new contracts and attempts to snipe eligible tokens.
 */
async function monitorAndSnipe() {
    console.log('\n[Monitor] Starting monitoring for new contracts on MovePump...');

    // Fetch new contracts from MovePump
    const newContracts = await getNewContractsFromMovePump();

    if (newContracts.length === 0) {
        console.log('[Monitor] No new contracts found at this time.');
        return;
    }

    // Process each new contract
    for (const contractAddress of newContracts) {
        await processContract(contractAddress);
    }
}

/**
 * Initializes and starts the sniper bot.
 */
async function initBot() {
    console.log('Initializing Sniper Bot...');

    // Start the monitoring loop
    setInterval(async () => {
        try {
            await monitorAndSnipe();
        } catch (error) {
            console.error('Error during monitoring and sniping:', error);
        }
    }, parseInt(process.env.CHECK_INTERVAL || '60000', 10)); // Default to 60 seconds

    // Optionally, perform an initial check immediately
    await monitorAndSnipe();
}

// --------------------- Start the Bot ---------------------

initBot().catch(error => {
    console.error('Bot initialization failed:', error);
    process.exit(1);
});
```

---

## **Step-by-Step Guide to Launch the Sniper Bot**

### **1. Prerequisites**

Before you begin, ensure you have the following installed on your system:

- **Node.js**: Version 14 or higher. [Download Node.js](https://nodejs.org/)
- **npm**: Comes bundled with Node.js.
- **TypeScript**: We'll use `ts-node` to run TypeScript files without manual compilation.

### **2. Set Up the Project Directory**

1. **Create a New Directory for the Bot**

    ```bash
    mkdir sui-sniper-bot
    cd sui-sniper-bot
    ```

2. **Initialize a New Node.js Project**

    ```bash
    npm init -y
    ```

### **3. Install Required Dependencies**

Install the necessary packages using `npm`:

```bash
npm install @mysten/sui.js axios dotenv
npm install -D typescript ts-node @types/node
```

- `@mysten/sui.js`: Sui blockchain SDK.
- `axios`: For making HTTP requests.
- `dotenv`: For managing environment variables.
- `typescript`, `ts-node`, `@types/node`: For TypeScript support and running TypeScript files directly.

### **4. Configure TypeScript**

Initialize a TypeScript configuration file:

```bash
npx tsc --init
```

This will generate a `tsconfig.json` file. You can use the default settings or customize them as needed. For this bot, the default configuration is sufficient.

### **5. Create the Sniper Bot File**

1. **Create `sniperBot.ts`**

    Create a new file named `sniperBot.ts` in the project directory and paste the finalized code provided above into it.

    ```bash
    touch sniperBot.ts
    ```

    Then, open `sniperBot.ts` in your preferred code editor and paste the provided code.

### **6. Set Up Environment Variables**

1. **Create a `.env` File**

    Create a `.env` file in the project directory to store all sensitive information and configurations.

    ```bash
    touch .env
    ```

2. **Populate the `.env` File**

    Open the `.env` file and add the following configurations. Replace placeholder values with your actual credentials and desired settings.

    ```env
    # Private Key for Sui Wallet (Hex string without 0x prefix)
    PRIVATE_KEY=your_private_key_here

    # RPC Endpoint
    RPC_URL=https://fullnode.devnet.sui.io

    # Inside API Configuration
    INSIDE_API_URL=https://api.inside.io/getContractScore
    INSIDE_API_KEY=your_inside_api_key_here

    # MovePump API Configuration
    MOVEPUMP_API_URL=https://api.movepump.dex/getNewContracts
    MOVEPUMP_API_KEY=your_movepump_api_key_here

    # TweetScout API Configuration
    TWEETSCOUT_API_URL=https://api.tweetscout.io/getTwitterScore
    TWEETSCOUT_API_KEY=your_tweetscout_api_key_here

    # Trading Score Thresholds
    SCORE_THRESHOLD=80
    TWITTER_SCORE_THRESHOLD=50

    # Trading Parameters
    SLIPPAGE=0.25             # 25% Slippage
    MIN_LIQUIDITY=10000       # Minimum Liquidity of 10K
    BUY_AMOUNT_SUI=50         # Buy Amount of 50 SUI
    CHECK_INTERVAL=60000      # Check every 60,000 ms (60 seconds)

    # DEX API Keys (if required)
    BLUEMOVE_API_KEY=your_bluemove_api_key_here
    CETUS_API_KEY=your_cetus_api_key_here
    ```

    **Notes:**

    - **Private Key**: Ensure your private key is a hex string without the `0x` prefix. **Never share your private key.**
    - **API Endpoints and Keys**: Replace all placeholder URLs and API keys with actual values provided by the respective services.
    - **Trading Parameters**: Adjust `SLIPPAGE`, `MIN_LIQUIDITY`, and `BUY_AMOUNT_SUI` as per your strategy. The current settings are:
        - **Slippage**: 25%
        - **Minimum Liquidity**: 10,000
        - **Buy Amount**: 50 SUI
    - **Check Interval**: The bot checks for new contracts every 60 seconds. You can adjust `CHECK_INTERVAL` as needed.

3. **Secure the `.env` File**

    To prevent accidental exposure of sensitive information, ensure that the `.env` file is added to `.gitignore`.

    ```bash
    echo ".env" >> .gitignore
    ```

### **7. Compile and Run the Bot**

You have two options to run the bot: using `ts-node` for direct execution or compiling it to JavaScript and running with `node`.

#### **Option 1: Using `ts-node`**

1. **Run the Bot**

    ```bash
    npx ts-node sniperBot.ts
    ```

    **Note**: Ensure that `ts-node` is installed as a development dependency (`-D`) as shown in the installation step.

#### **Option 2: Compiling to JavaScript and Running with Node**

1. **Compile TypeScript to JavaScript**

    ```bash
    npx tsc sniperBot.ts
    ```

    This will generate a `sniperBot.js` file in the same directory.

2. **Run the Compiled JavaScript File**

    ```bash
    node sniperBot.js
    ```

### **8. Monitoring and Logs**

Once the bot is running, it will output logs to the console, indicating its activities, such as:

- Fetching new contracts from MovePump.
- Evaluating each contract's score, Twitter score, bonding curve, and liquidity.
- Executing trades on supported DEXs.
- Any errors or warnings encountered during operation.

**Example Console Output:**

```
Initializing Sniper Bot...

[Monitor] Starting monitoring for new contracts on MovePump...
Fetched 3 new contracts from MovePump.

[Process] Evaluating contract: 0xABC123...
Contract score for 0xABC123: 85
Twitter score for 0xABC123: 60
Bonding curve for 0xABC123 is 100%.
Liquidity for 0xABC123 meets the minimum requirement.
All checks passed for 0xABC123. Proceeding to snipe.

Attempting to execute trade on BlueMove for contract 0xABC123...
Trade executed successfully on BlueMove for contract 0xABC123

Attempting to execute trade on Cetus for contract 0xABC123...
Trade executed successfully on Cetus for contract 0xABC123

Attempting to execute trade on MovePump for contract 0xABC123...
Trade executed successfully on MovePump for contract 0xABC123

[Monitor] Starting monitoring for new contracts on MovePump...
No new contracts found at this time.
```

### **9. Security Best Practices**

- **Protect Your Private Key**: 
    - **Never expose your private key** in code repositories, logs, or any public places.
    - Consider using secret management services or environment variable managers for enhanced security.
  
- **Secure API Keys**:
    - Restrict API keys to only necessary permissions.
    - Regularly rotate API keys and monitor their usage.

- **Use `.gitignore`**:
    - Ensure your `.env` file and other sensitive files are listed in `.gitignore` to prevent accidental commits.

    ```gitignore
    # .gitignore
    .env
    node_modules/
    dist/
    ```

- **Regular Audits**:
    - Periodically review your code and configurations for potential vulnerabilities or misconfigurations.

### **10. Testing the Bot**

Before deploying the bot with real funds, thoroughly test it in a **development or testnet environment**:

1. **Use Test Credentials**: Replace your private key and API keys with test equivalents provided by the respective services.

2. **Dry-Run Mode**: Implement a simulation mode where the bot logs intended actions without executing actual trades. This helps verify logic without financial risk.

    ```typescript
    const DRY_RUN = process.env.DRY_RUN === 'true';

    // Modify executeTrade function to handle dry run
    async function executeTrade(dex: { name: string; endpoint: string; apiKey?: string }, contractAddress: string) {
        console.log(`\nAttempting to execute trade on ${dex.name} for contract ${contractAddress}...`);

        if (DRY_RUN) {
            console.log(`[Dry Run] Trade simulated on ${dex.name} for contract ${contractAddress}`);
            return;
        }

        // Existing trade execution logic...
    }
    ```

    Update `.env`:

    ```env
    DRY_RUN=true
    ```

3. **Monitor Logs**: Ensure all functionalities work as expected without executing real trades.

### **11. Advanced Enhancements (Optional)**

To further enhance the bot's capabilities and robustness, consider implementing the following features:

- **Real-Time Event Listening**:
    - Use WebSockets or similar technologies to listen for real-time events from MovePump or the blockchain, enabling faster reaction times compared to periodic polling.

- **Dynamic Trading Strategies**:
    - Incorporate more sophisticated trading strategies, such as arbitrage or algorithmic trading based on market indicators.

- **Notifications**:
    - Integrate notification systems (e.g., email, SMS, Telegram) to receive alerts about significant bot activities or errors.

- **Database Integration**:
    - Use a database to store transaction history, logs, and performance metrics for analysis and auditing purposes.

- **Error Retries and Fallbacks**:
    - Implement retry mechanisms for transient failures and ensure that failures in one part of the bot do not halt the entire operation.

### **12. Compliance and Ethics**

- **Legal Compliance**:
    - Ensure that your bot complies with all relevant laws and regulations in your jurisdiction and the jurisdictions of the DEXs involved.

- **Ethical Considerations**:
    - Consider the ethical implications of automated trading, such as market impact and fairness.

---

## **Final Notes**

Building a sniper bot for the blockchain is a complex task that involves interacting with multiple APIs, handling asynchronous operations, and ensuring robust error handling and security measures. The provided code serves as a comprehensive foundation, integrating key functionalities such as contract fetching, bonding curve validation, contract scoring, Twitter score evaluation, liquidity checks, and trade execution across multiple DEXs.

**Always prioritize security, compliance, and thorough testing to ensure the bot operates reliably and safely in live trading scenarios.**

**Disclaimer**: Automated trading involves significant financial risks. Ensure you fully understand the risks and have appropriate safeguards in place before deploying the bot with real assets.

---
