https://monadvision.com/address/0x9F93da6f9202fF6566b35D4D1Da6e1bb4a64EB56?tab=Contract

合约的区块链地址


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title STEADYNADS - Fair Launch Token with Anti-Bot Protection
 * @dev Token with fair launch mining mechanism for community
 * Total Supply: 1,000,000,000 STEADYNADS
 * Mintable: 900,000,000 STEADYNADS
 * Treasury Reserve: 100,000,000 STEADYNADS
 * 
 */

contract STEADYNADS {
    string public constant name = "STEADYNADS";
    string public constant symbol = "SNADS";
    uint8 public constant decimals = 18;
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 10**18;
    uint256 public constant MINTABLE_SUPPLY = 900_000_000 * 10**18;
    uint256 public constant TREASURY_RESERVE = 100_000_000 * 10**18;
    uint256 public constant MINT_AMOUNT = 5000 * 10**18;
    uint256 public constant MINT_COST = 1 ether; // 1 MON
    uint256 public constant COOLDOWN_PERIOD = 10; // 10 seconds
    uint256 public constant MAX_MINTS_PER_WALLET = 1000;

    address public immutable treasury;
    uint256 public totalMinted;
    uint256 public launchTime;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    mapping(address => uint256) public lastMintTime;
    mapping(address => uint256) public mintCount;
    mapping(address => uint256) public lastMintBlock;
    
    // Anti-bot mappings
    mapping(address => bool) public isBlacklisted;
    mapping(bytes32 => bool) public usedNonces;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Minted(address indexed minter, uint256 amount, uint256 cost);
    event Blacklisted(address indexed account);

    modifier noContract() {
        require(msg.sender == tx.origin, "Contracts not allowed");
        require(!_isContract(msg.sender), "Contract addresses blocked");
        _;
    }

    modifier notBlacklisted() {
        require(!isBlacklisted[msg.sender], "Address is blacklisted");
        _;
    }

    constructor() {
        treasury = 0x949A2283494A2f35C4700a272BA87126417198E1;
        launchTime = block.timestamp;
        
        // Mint treasury reserve directly to treasury address
        balanceOf[treasury] = TREASURY_RESERVE;
        totalMinted = TREASURY_RESERVE;
        emit Transfer(address(0), treasury, TREASURY_RESERVE);
    }

    /**
     * @dev Check if address is a contract
     */
    function _isContract(address account) internal view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }

    /**
     * @dev Anti-bot: Detect suspicious patterns and blacklist
     * FIXED: Only blocks multiple mints in SAME block, not consecutive blocks
     */
    function _checkAndBlacklistBot(address account) internal {
        // If minting in same block as last mint = bot behavior
        if (lastMintBlock[account] == block.number && mintCount[account] > 0) {
            isBlacklisted[account] = true;
            emit Blacklisted(account);
            revert("Multiple mints in same block detected");
        }
        
        // Update last mint block
        lastMintBlock[account] = block.number;
    }

    /**
     * @dev Function to mine/mint tokens with anti-bot protection
     * Requires payment of 1 MON which will be sent to treasury
     * FIXED: Removed problematic first-mint block check
     */
    function mint() external payable noContract notBlacklisted {
        require(msg.value == MINT_COST, "Must send exactly 1 MON");
        require(totalMinted + MINT_AMOUNT <= TOTAL_SUPPLY, "Max supply reached");
        require(
            block.timestamp >= lastMintTime[msg.sender] + COOLDOWN_PERIOD,
            "Cooldown period not passed"
        );
        require(mintCount[msg.sender] < MAX_MINTS_PER_WALLET, "Max mints per wallet reached");

        // Anti-bot check - will revert if minting in same block
        _checkAndBlacklistBot(msg.sender);

        // Update state
        lastMintTime[msg.sender] = block.timestamp;
        mintCount[msg.sender]++;
        totalMinted += MINT_AMOUNT;
        balanceOf[msg.sender] += MINT_AMOUNT;

        // Transfer MON to treasury
        (bool success, ) = treasury.call{value: msg.value}("");
        require(success, "Transfer to treasury failed");

        emit Minted(msg.sender, MINT_AMOUNT, msg.value);
        emit Transfer(address(0), msg.sender, MINT_AMOUNT);
    }

    /**
     * @dev Advanced mint with nonce for additional security (optional)
     * Prevents replay attacks and adds extra randomness
     */
    function mintWithNonce(bytes32 nonce) external payable noContract notBlacklisted {
        require(!usedNonces[nonce], "Nonce already used");
        require(msg.value == MINT_COST, "Must send exactly 1 MON");
        require(totalMinted + MINT_AMOUNT <= TOTAL_SUPPLY, "Max supply reached");
        require(
            block.timestamp >= lastMintTime[msg.sender] + COOLDOWN_PERIOD,
            "Cooldown period not passed"
        );
        require(mintCount[msg.sender] < MAX_MINTS_PER_WALLET, "Max mints per wallet reached");

        // Anti-bot check
        _checkAndBlacklistBot(msg.sender);

        // Mark nonce as used
        usedNonces[nonce] = true;

        // Update state
        lastMintTime[msg.sender] = block.timestamp;
        mintCount[msg.sender]++;
        totalMinted += MINT_AMOUNT;
        balanceOf[msg.sender] += MINT_AMOUNT;

        // Transfer MON to treasury
        (bool success, ) = treasury.call{value: msg.value}("");
        require(success, "Transfer to treasury failed");

        emit Minted(msg.sender, MINT_AMOUNT, msg.value);
        emit Transfer(address(0), msg.sender, MINT_AMOUNT);
    }

    /**
     * @dev Check if an address can mint now
     */
    function canMint(address account) external view returns (bool) {
        if (isBlacklisted[account]) return false;
        if (_isContract(account)) return false;
        if (totalMinted + MINT_AMOUNT > TOTAL_SUPPLY) return false;
        if (mintCount[account] >= MAX_MINTS_PER_WALLET) return false;
        if (block.timestamp < lastMintTime[account] + COOLDOWN_PERIOD) return false;
        return true;
    }

    /**
     * @dev Time remaining until next mint is available (in seconds)
     */
    function timeUntilNextMint(address account) external view returns (uint256) {
        if (block.timestamp >= lastMintTime[account] + COOLDOWN_PERIOD) {
            return 0;
        }
        return (lastMintTime[account] + COOLDOWN_PERIOD) - block.timestamp;
    }

    /**
     * @dev Remaining mintable supply globally
     */
    function remainingMintableSupply() external view returns (uint256) {
        if (totalMinted >= TOTAL_SUPPLY) return 0;
        return TOTAL_SUPPLY - totalMinted;
    }

    /**
     * @dev Remaining mints available for a specific wallet
     */
    function remainingMintsForWallet(address account) external view returns (uint256) {
        if (mintCount[account] >= MAX_MINTS_PER_WALLET) return 0;
        return MAX_MINTS_PER_WALLET - mintCount[account];
    }

    // Standard ERC-20 Functions
    function totalSupply() external pure returns (uint256) {
        return TOTAL_SUPPLY;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        
        emit Transfer(from, to, amount);
        return true;
    }
}
