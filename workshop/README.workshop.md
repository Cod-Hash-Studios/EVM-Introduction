# Workshop: Création d'un Token ERC20 et d'une DAO Personnalisée
> De l'ERC20 à la gouvernance DAO avec OpenZeppelin et Aragon, et plus encore!

Ce workshop vous guidera dans la création d'une DAO complète avec votre propre token de gouvernance. Vous apprendrez à utiliser Forge, OpenZeppelin et Aragon pour construire une infrastructure de gouvernance décentralisée.
Des references seront données pour aller plus loin sur differents sujets.

## Prérequis

- Terminal/ligne de commande
- Git installé
- Node.js (v16+) et npm/yarn installés

## Installation & Setup

### 1. Installation de Foundry/Forge

```bash
# Installation de Foundry (inclut Forge)
curl -L https://foundry.paradigm.xyz | bash

# Rechargez votre terminal puis lancez
foundryup

# Vérifiez l'installation
forge --version
```

### 2. Création du Projet

```bash
# Créez et accédez au dossier du projet
mkdir my-custom-dao && cd my-custom-dao

# Initialisez un projet Forge
forge init

# Initialisez npm pour gérer les dépendances JS
npm init -y

# Installez ts + node pour run des scripts typescript en plus des scripts forge (optionnel)
npm install typescript ts-node --save-dev
npm i --save-dev @types/node
```

### 3. Installation des Dépendances

```bash
# Installation des contracts OpenZeppelin - Standards Smart Contracts
forge install OpenZeppelin/openzeppelin-contracts --no-commit

# Installation Dotenv - gestion des secrets
npm install dotenv
```

### 4. Configuration du Projet

#### 4.1 Créez/modifiez le fichier `foundry.toml`:
```toml
[profile.default]
src = 'src'
out = 'out'
libs = ['lib']
remappings = [
    '@openzeppelin/=lib/openzeppelin-contracts/'
]
```

#### 4.2 Créez/modifiez le fichier `tsconfig.json`:
```json
   {
     "compilerOptions": {
       "target": "ES6",
       "module": "commonjs",
       "strict": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "forceConsistentCasingInFileNames": true,
				"resolveJsonModule": true
     },
     "include": ["script/**/*.ts"]
   }
```
## Structure du Workshop

### Partie 1: Token ERC20
- Création du token
- Tests et déploiement

### Partie 2: Setup DAO
- Integration testnet Token a une DAO via l'UI Aragon

### Optionnel: créer un custom plugin
- Intégration Aragon OSx 
- Configuration du plugin de vote
- Paramétrage de la gouvernance

### Partie 3: Pour aller plus loin
- Intégration d'un contract de Lock
- Intégration d'un contract de Reward
- Intégration d'un contract de Airdrop

## Guide Étape par Étape

### 1. Création du Token
src/Token.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// Importing ERC20 and ERC20Permit audited standardsfrom OpenZeppelin
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

// Extending our contract with the ERC20 and ERC20Permit standards
contract GovernanceToken is ERC20, ERC20Permit {
    address owner;
		
		// Initialisation of the contract with the name and symbol
    constructor(string memory name, string memory symbol)
        ERC20(name, symbol)
        ERC20Permit(name)
    {
        // Setting the owner of the contract to the deployer
        owner = msg.sender;
    }

    // Function to mint new tokens
    function mint(address to, uint256 amount) public {
        // Only the owner can mint new tokens
        require(msg.sender == owner, "Only owner can mint");
        _mint(to, amount);
    }
}

```

#### 1.1 Creation des tests

test/TestToken.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {GovernanceToken} from "../src/Token.sol";

contract GovernanceTokenTest is Test {
    GovernanceToken public token;

    // Initialisation function, run before each test
    function setUp() public {
        token = new GovernanceToken("MyDaoToken", "DAO");
    }

		// Basic Mint Test 
    function test_Mint() public {
        token.mint(address(this), 1000);
        assertEq(token.balanceOf(address(this)), 1000);
    }

    // Fuzzing Mint Test - Test multiple iterations with random values
    function testFuzz_Mint(uint256 x) public {
        token.mint(address(this), x);
        assertEq(token.balanceOf(address(this)), x);
    }
}
```

#### 1.2 Utils Script

script/deployToken.s.sol
```solidity
import {Script, console} from "forge-std/Script.sol";
import {GovernanceToken} from "../src/Token.sol";
contract TokenScript is Script {
    GovernanceToken public token;

    function setUp() public {}

    function run() public {
				vm.startBroadcast();

        token = new GovernanceToken("TestToken", "TEST");

        vm.stopBroadcast();


    }
}
```
script/createWallet.ts
```ts
import { ethers } from 'ethers';

// Generate a wallet with a mnemonic
const wallet = ethers.Wallet.createRandom();

// Access the mnemonic phrase
console.log(`mnemo:"${wallet.mnemonic.phrase}",`);
console.log(`address:"${wallet.address}",`);
console.log(`privateKey:"${wallet.privateKey}"`);

```

### 2. Test Token

```bash
# Compilation
forge build

# Tests
forge test
```

#### 3. Deploiement
```bash
# Compilation
forge build

# Tests
forge test

# Creation d'un test dev wallet (optionnel)
npx ts-node script/createWallet.ts

# Recuperer un RPC public 
https://chainlist.org/chain/11155111
# Ou prive 
https://www.alchemy.com/


# Fund le test dev wallet avec des fake ETH par un faucet 
https://faucets.chain.link/sepolia


# Deploiement (testnet)

## par ligne de cmd
forge create src/Token.sol:GovernanceToken --rpc-url https://ethereum-sepolia-rpc.publicnode.com --private-key <PRIVATE_KEY> --constructor-args "MyDaoToken" "DAO" --broadcast -vvv
## par script
forge script script/deployToken.s.sol --rpc-url https://ethereum-sepolia-rpc.publicnode.com --private-key <PRIVATE_KEY> --broadcast

# Verification du contrat
forge verify-contract --chain-id 11155111 <ADRESSE_DU_CONTRAT> src/Token.sol:GovernanceToken --etherscan-api-key <ETHERSCAN_API_KEY> --constructor-args $(cast abi-encode "constructor(string,string)" "MyDaoToken" "DAO") --watch
```

### 4. Setup DAO
une fois le token deployé, vous pouvez setup une DAO avec le token.
rendez vous sur https://app.aragon.org/! 

Pour les plus curieux, vous pouvez deployer par code une DAO personnalisée (avec des plugins custom) avec votre token.
https://devs.aragon.org/

## Commandes Utiles

```bash
# Compilation
forge build

# Tests
forge test

# Déploiement
forge create

# Clean
forge clean
```

## Architecture du Projet
```
my-custom-dao/
├── src/
│   └── Token.sol           # Token de gouvernance
├── test/                   # Tests
│   └── TestToken.t.sol     # Tests du token
├── script/                 # Scripts de déploiement
│   ├── deployToken.s.sol   # Deploiement du token
│   └── createWallet.ts     # Creation d'un wallet de test
├── frontend/              # Interface (optionnel)
├── package.json            # Gestion des dépendances JS
└── foundry.toml            # Configuration Foundry
```

## Troubleshooting

### Problèmes Courants

1. **Erreur de compilation OpenZeppelin**
   ```bash
   forge remappings > remappings.txt
   forge build
   ```

## Pour aller plus loin

- Ajout de mécanismes de vote personnalisés
- Intégration de timelock
- Déploiement pool uniswap sur testnet
- Creation de mini jeux
- Setup frontend 
- https://solidity-by-example.org/

## Ressources

- [Documentation Forge](https://book.getfoundry.sh/)
- [OpenZeppelin Wizard](https://wizard.openzeppelin.com/)
- [Documentation Aragon](https://devs.aragon.org/)
- [Aragon Deployed Contracts](https://github.com/aragon/osx/blob/b0ca2fb9bf09b96f2d9ddf0a2121f5ff2735bc3e/packages/contracts/Releases.md)
- [Solidity by example](https://solidity-by-example.org/)
- [Aavegotchi](https://github.com/aavegotchi/aavegotchi-contracts)
- [Diamond Framework](https://gemforge.xyz/getting-started/)

## Support

- Discord Workshop: https://discord.gg/bfu7SQGCvu

---

*Note: Ce workshop est destiné à des fins éducatives. En production, des audits de sécurité supplémentaires seraient nécessaires.*