# 🎮 On-Chain Game Scores Deployment Guide

Complete guide for deploying the GameScores smart contract and integrating it with your game.

## 📋 Prerequisites

All tools are already installed! Verify:

```bash
scarb --version        # Should show v2.16.1 or higher
snforge --version      # Should show v0.57.0 or higher
sncast --version       # Should show v0.57.0 or higher
```

## 🚀 Deployment Steps

### Step 1: Get Testnet STRK Tokens

1. Visit the Starknet Sepolia faucet: https://starknet-faucet.vercel.app/
2. Enter your wallet address
3. Request test STRK tokens (needed for contract deployment)

### Step 2: Set Up Environment Variables

Create a `.env` file in the project root:

```bash
cp .env.example .env
```

Edit `.env` and add your deployer wallet details:

```bash
STARKNET_RPC_URL=https://starknet-sepolia.public.blastapi.io/rpc/v0_7
DEPLOYER_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
DEPLOYER_ADDRESS=0xYOUR_ADDRESS_HERE
```

**⚠️ IMPORTANT:** Never commit `.env` to git! It's already in `.gitignore`.

### Step 3: Declare the Contract

Navigate to the contract directory and declare:

```bash
cd game_scores

# Load environment variables
source ../.env

# Declare the contract (registers code on-chain)
sncast --url $STARKNET_RPC_URL \
  declare \
  --contract-name GameScores \
  --account $DEPLOYER_ADDRESS \
  --private-key $DEPLOYER_PRIVATE_KEY
```

**Save the `class_hash` from the output!** You'll need it for deployment.

Example output:
```
class_hash: 0x0123abc...
```

### Step 4: Deploy the Contract

Deploy an instance using the class hash:

```bash
sncast --url $STARKNET_RPC_URL \
  deploy \
  --class-hash 0xYOUR_CLASS_HASH_FROM_STEP_3 \
  --account $DEPLOYER_ADDRESS \
  --private-key $DEPLOYER_PRIVATE_KEY
```

**Save the `contract_address` from the output!** This is your deployed contract.

Example output:
```
contract_address: 0x0456def...
```

### Step 5: Verify Deployment

Test a read call to ensure the contract works:

```bash
sncast --url $STARKNET_RPC_URL \
  call \
  --contract-address 0xYOUR_CONTRACT_ADDRESS \
  --function get_global_record \
  --calldata ""
```

Should return all zeros (no scores yet).

### Step 6: Configure Frontend

Update `frontend/.env`:

```bash
cd ../frontend
cp .env.example .env
```

Edit `frontend/.env`:

```bash
REACT_APP_STARKNET_NETWORK=sepolia
REACT_APP_RPC_URL=https://starknet-sepolia.public.blastapi.io/rpc/v0_7
REACT_APP_GAME_SCORES_CONTRACT=0xYOUR_CONTRACT_ADDRESS_FROM_STEP_4
```

### Step 7: Install Frontend Dependencies (Already Done!)

Dependencies are already installed:
- ✅ starkzap
- ✅ @cartridge/controller
- ✅ starknet

### Step 8: Integrate with Your Game

Add the wallet connection and leaderboard to your game:

```jsx
import { useOnChainScores } from './hooks/useOnChainScores';
import { WalletConnect } from './components/WalletConnect';
import { Leaderboard } from './components/Leaderboard';

function YourGameComponent() {
  const {
    connectWallet,
    disconnectWallet,
    submitScore,
    loadGlobalRecord,
    loadPlayerBest,
    loadTotalPlayers,
    isConnected,
    playerAddress,
    isSubmitting,
    error,
    globalRecord,
  } = useOnChainScores();

  const [playerBest, setPlayerBest] = useState(null);
  const [totalPlayers, setTotalPlayers] = useState(0);

  // Call submitScore when player completes a level or game
  const handleGameComplete = async (level, deaths, coins) => {
    const score = calculateScore(level, deaths, coins);
    
    if (isConnected) {
      const result = await submitScore({ 
        score, 
        level, 
        deaths, 
        coins 
      });
      
      if (result) {
        console.log('Score saved on-chain:', result.explorerUrl);
      }
    }
  };

  // Example score calculation
  const calculateScore = (level, deaths, coins) => {
    return Math.max(0, (level * 1000) - (deaths * 50) + (coins * 10));
  };

  return (
    <div>
      {/* Wallet Connection */}
      <WalletConnect
        isConnected={isConnected}
        playerAddress={playerAddress}
        onConnect={connectWallet}
        onDisconnect={disconnectWallet}
        error={error}
      />

      {/* Leaderboard */}
      <Leaderboard
        globalRecord={globalRecord}
        playerBest={playerBest}
        totalPlayers={totalPlayers}
        onLoadGlobalRecord={loadGlobalRecord}
        onLoadPlayerBest={async () => {
          const best = await loadPlayerBest();
          setPlayerBest(best);
        }}
        onLoadTotalPlayers={async () => {
          const total = await loadTotalPlayers();
          setTotalPlayers(total);
        }}
        isConnected={isConnected}
      />

      {/* Your game UI here */}
    </div>
  );
}
```

## 🔍 Verify on Block Explorer

Once deployed, view your contract on Starkscan:

**Sepolia:** https://sepolia.starkscan.co/contract/YOUR_CONTRACT_ADDRESS

You can:
- View all transactions
- See score submissions
- Monitor global records
- Check events (NewHighScore, GlobalRecordBroken)

## 🎯 How It Works

### Gasless Transactions

Players connect using **Cartridge Controller**, which provides:
- ✅ Social login (Google, Twitter, passkey)
- ✅ No seed phrases needed
- ✅ All transactions are FREE for players (gasless)
- ✅ Automatic transaction sponsorship

### Score Submission

When a player completes a game:
1. `submitScore()` is called with score, level, deaths, coins
2. Contract only updates if score is higher than personal best
3. Global record automatically updated if beaten
4. Events emitted for real-time tracking
5. Player pays zero gas fees!

### Contract Features

- **Per-player best scores**: Each wallet stores their highest score
- **Global leaderboard**: Tracks the all-time world record holder
- **Player statistics**: Stores level, deaths, coins for context
- **Total player count**: Tracks unique players
- **Event emission**: Real-time updates via blockchain events

## 🧪 Testing

### Test Score Submission

```bash
# Submit a test score using sncast
sncast --url $STARKNET_RPC_URL \
  invoke \
  --contract-address $GAME_SCORES_CONTRACT \
  --function submit_score \
  --calldata 1000 5 10 42 \
  --account $DEPLOYER_ADDRESS \
  --private-key $DEPLOYER_PRIVATE_KEY

# Read it back
sncast --url $STARKNET_RPC_URL \
  call \
  --contract-address $GAME_SCORES_CONTRACT \
  --function get_player_best \
  --calldata $DEPLOYER_ADDRESS
```

### Frontend Testing

```bash
cd frontend
npm start
```

1. Click "Connect Wallet"
2. Sign in with Cartridge (Google/Twitter/passkey)
3. Play a game
4. Watch your score appear on-chain!
5. Check the leaderboard updates

## 📦 Project Structure

```
Koool_Shit/
├── game_scores/              # Cairo contract
│   ├── Scarb.toml
│   ├── src/lib.cairo        # GameScores smart contract
│   └── target/              # Compiled artifacts
├── frontend/
│   ├── src/
│   │   ├── starkzap/
│   │   │   ├── config.ts    # Contract address & network
│   │   │   ├── wallet.ts    # Wallet connection
│   │   │   └── scores.ts    # Score submission & reading
│   │   ├── hooks/
│   │   │   └── useOnChainScores.ts  # React hook
│   │   └── components/
│   │       ├── WalletConnect.jsx    # Wallet UI
│   │       └── Leaderboard.jsx      # Leaderboard UI
│   └── .env                 # Frontend config (gitignored)
├── .env                     # Deployment config (gitignored)
└── DEPLOYMENT_GUIDE.md      # This file
```

## 🚨 Troubleshooting

### "Account not deployed" error
- Run with `deploy: "if_needed"` in `sdk.onboard()` (already configured)
- Or manually deploy the account first

### "Insufficient balance" error
- Get more Sepolia STRK from the faucet
- Wait a few minutes for faucet transaction to confirm

### "Class hash not found" error
- Wait 30 seconds after declare
- Sepolia testnet can be slow
- Retry the deploy command

### Frontend can't connect
- Check `.env` has correct contract address
- Verify network is set to "sepolia"
- Clear browser cache and reload

### Score not updating
- Contract only stores personal bests
- If new score ≤ old score, it won't update
- This is expected behavior!

## 🎉 Next Steps

### Production Deployment (Mainnet)

1. Change network to `mainnet` in configs
2. Get real STRK tokens (not testnet)
3. Redeploy contract to mainnet
4. Update frontend `.env` with mainnet contract address
5. **Consider applying for Starknet Propulsion Program** (up to $1M in gas subsidies)

### Apply for Gas Subsidies

For production, apply for the **Starknet Propulsion Program**:
- Website: https://docs.avnu.fi/docs/paymaster/propulsion-program
- Up to $1M in sponsored transactions
- Perfect for games with many players
- Completely free for your players

## 📚 Resources

- **StarkZap Docs:** https://docs.starknet.io/build/starkzap/overview
- **Cartridge Controller:** https://docs.starknet.io/build/starkzap/integrations/cartridge-controller
- **Cairo Book:** https://book.cairo-lang.org/
- **Starknet Book:** https://book.starknet.io/
- **Sepolia Faucet:** https://starknet-faucet.vercel.app/
- **Sepolia Explorer:** https://sepolia.starkscan.co

## ✅ Deployment Checklist

- [ ] Get Sepolia STRK from faucet
- [ ] Create `.env` with deployer credentials
- [ ] Build contract: `scarb build`
- [ ] Declare contract with `sncast declare`
- [ ] Save class_hash
- [ ] Deploy contract with `sncast deploy`
- [ ] Save contract_address
- [ ] Verify deployment with test call
- [ ] Update `frontend/.env` with contract address
- [ ] Test wallet connection in browser
- [ ] Submit test score
- [ ] Verify score on Starkscan
- [ ] Integrate with game UI
- [ ] Test complete game flow
- [ ] 🎉 Launch!

---

**Need help?** Check the troubleshooting section or open an issue!
