## Sepolia Testnet With CustomGasToken

Explainer : [CustomGasToken](https://docs.optimism.io/stack/protocol/features/custom-gas-token)

Make sure to have more than 1 ETH just in case the network has a gasFee increase.

Update the deploy config in `contracts-bedrock/deploy-config` with new fields: `useCustomGasToken` and `customGasTokenAddress`

Set `useCustomGasToken` to `true`. If you set `useCustomGasToken` to `false` (it defaults this way), then it will use ETH as the gas paying token.

Set `customGasTokenAddress` to the contract address of an L1 ERC20 token you wish to use as the gas token on your L2. The ERC20 should already be deployed before trying to spin up the custom gas token chain.

### The Custom Gas Token being set must meet the following criteria:
- must be a valid ERC-20 token
- the number of decimals on the token MUST be exactly 18
- the name of the token MUST be less than or equal to 32 bytes
- symbol MUST be less than or equal to 32 bytes.
- must not be yield-bearing
- cannot be rebasing or have a transfer fee
- must be transferrable only via a call to the token address itself
- must only be able to set allowance via a call to the token address itself
- must not have a callback on transfer, and more generally a user must not be able to make a transfer to themselves revert
- a user must not be able to make a transfer have unexpected side effects

```bash
git clone https://github.com/layeronly/optimism
cd optimism
git checkout op-contracts/v2.0.0-beta.2
```

```bash
pnpm install 
make op-node op-batcher op-proposer
pnpm build
```

```bash
cd ~
git clone https://github.com/ethereum-optimism/op-geth.git
cd op-geth
make op-geth
```

Setup ENV VARS
```bash
cd ~/optimism
cp .envrc.example .envrc
direnv allow
```

Output : 
```bash
direnv: loading ~/optimism/.envrc                                                            
direnv: export +DEPLOYMENT_CONTEXT +ETHERSCAN_API_KEY +GS_ADMIN_ADDRESS +GS_ADMIN_PRIVATE_KEY +GS_BATCHER_ADDRESS +GS_BATCHER_PRIVATE_KEY +GS_PROPOSER_ADDRESS +GS_PROPOSER_PRIVATE_KEY +GS_SEQUENCER_ADDRESS +GS_SEQUENCER_PRIVATE_KEY +IMPL_SALT +L1_RPC_KIND +L1_RPC_URL +PRIVATE_KEY +TENDERLY_PROJECT +TENDERLY_USERNAME
```

Generate & Deploy Contract L1 : 
```bash
DEPLOYMENT_OUTFILE=/root/optimism/packages/contracts-bedrock/deployments/11155111/artifact.json \
DEPLOY_CONFIG_PATH=/root/optimism/packages/contracts-bedrock/deploy-config/getting-started.json \
  forge script scripts/Deploy.s.sol:Deploy \
  --broadcast --private-key $GS_ADMIN_PRIVATE_KEY \
  --rpc-url $L1_RPC_URL
```

Generate the Required state-dump-***.json File :
```bash
CONTRACT_ADDRESSES_PATH=/root/optimism/packages/contracts-bedrock/deployments/11155111/artifact.json \
DEPLOY_CONFIG_PATH=/root/optimism/packages/contracts-bedrock/deploy-config/getting-started.json \
  forge script scripts/L2Genesis.s.sol:L2Genesis \
  --chain-id 11155111 --sig "runWithAllUpgrades()" \
  --private-key $GS_ADMIN_PRIVATE_KEY
```

Generate L2 Allocs :
```bash
CONTRACT_ADDRESSES_PATH=/root/optimism/packages/contracts-bedrock/deployments/11155111/artifact.json \
DEPLOY_CONFIG_PATH=/root/optimism/packages/contracts-bedrock/deploy-config/getting-started.json \
STATE_DUMP_PATH=/root/optimism/packages/contracts-bedrock/deployments/11155111/state-dump-231196.json \
  forge script scripts/L2Genesis.s.sol:L2Genesis \
  --sig 'runWithStateDump()' --chain-id $L2_CHAIN_ID
```

Generate L2 Genesis : 
```bash
op-node genesis l2 \
  --l1-rpc $L1_RPC_URL \
  --deploy-config /root/optimism/packages/contracts-bedrock/deploy-config/getting-started.json \
  --l2-allocs /root/optimism/packages/contracts-bedrock/deployments/11155111/state-dump-231196.json \
  --l1-deployments /root/optimism/packages/contracts-bedrock/deployments/11155111/artifact.json \
  --outfile.l2 /root/optimism/packages/contracts-bedrock/deployments/11155111/genesis-l2.json \
  --outfile.rollup /root/optimism/packages/contracts-bedrock/deployments/11155111/rollup.json
```