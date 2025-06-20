// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title NFTMarketplace
 * @dev A decentralized marketplace for buying and selling NFTs
 */
contract NFTMarketplace is ReentrancyGuard, Ownable {
    
    // Marketplace fee percentage (2.5%)
    uint256 public constant MARKETPLACE_FEE = 250; // 250 basis points = 2.5%
    uint256 public constant BASIS_POINTS = 10000;
    
    // Listing counter for unique listing IDs
    uint256 private _listingIdCounter;
    
    // Marketplace earnings
    uint256 public marketplaceEarnings;
    
    struct Listing {
        uint256 listingId;
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }
    
    // Mapping from listing ID to listing details
    mapping(uint256 => Listing) public listings;
    
    // Mapping from NFT contract and token ID to listing ID
    mapping(address => mapping(uint256 => uint256)) public nftToListing;
    
    // Events
    event NFTListed(
        uint256 indexed listingId,
        address indexed seller,
        address indexed nftContract,
        uint256 tokenId,
        uint256 price
    );
    
    event NFTPurchased(
        uint256 indexed listingId,
        address indexed buyer,
        address indexed seller,
        uint256 price
    );
    
    event ListingCancelled(uint256 indexed listingId);
    
    constructor() Ownable(msg.sender) {}
    
    /**
     * @dev List an NFT for sale on the marketplace
     * @param _nftContract Address of the NFT contract
     * @param _tokenId Token ID of the NFT
     * @param _price Price in wei
     */
    function listNFT(
        address _nftContract,
        uint256 _tokenId,
        uint256 _price
    ) external nonReentrant {
        require(_price > 0, "Price must be greater than zero");
        require(_nftContract != address(0), "Invalid NFT contract address");
        
        IERC721 nft = IERC721(_nftContract);
        require(nft.ownerOf(_tokenId) == msg.sender, "You don't own this NFT");
        require(
            nft.getApproved(_tokenId) == address(this) || 
            nft.isApprovedForAll(msg.sender, address(this)),
            "Marketplace not approved to transfer NFT"
        );
        require(
            nftToListing[_nftContract][_tokenId] == 0,
            "NFT already listed"
        );
        
        _listingIdCounter++;
        uint256 listingId = _listingIdCounter;
        
        listings[listingId] = Listing({
            listingId: listingId,
            seller: msg.sender,
            nftContract: _nftContract,
            tokenId: _tokenId,
            price: _price,
            active: true
        });
        
        nftToListing[_nftContract][_tokenId] = listingId;
        
        emit NFTListed(listingId, msg.sender, _nftContract, _tokenId, _price);
    }
    
    /**
     * @dev Purchase an NFT from the marketplace
     * @param _listingId ID of the listing
     */
    function purchaseNFT(uint256 _listingId) external payable nonReentrant {
        Listing storage listing = listings[_listingId];
        require(listing.active, "Listing not active");
        require(msg.value == listing.price, "Incorrect payment amount");
        require(msg.sender != listing.seller, "Cannot buy your own NFT");
        
        IERC721 nft = IERC721(listing.nftContract);
        require(
            nft.ownerOf(listing.tokenId) == listing.seller,
            "Seller no longer owns NFT"
        );
        
        // Calculate fees
        uint256 marketplaceFee = (listing.price * MARKETPLACE_FEE) / BASIS_POINTS;
        uint256 sellerAmount = listing.price - marketplaceFee;
        
        // Update marketplace earnings
        marketplaceEarnings += marketplaceFee;
        
        // Mark listing as inactive
        listing.active = false;
        nftToListing[listing.nftContract][listing.tokenId] = 0;
        
        // Transfer NFT to buyer
        nft.safeTransferFrom(listing.seller, msg.sender, listing.tokenId);
        
        // Transfer payment to seller
        (bool success, ) = payable(listing.seller).call{value: sellerAmount}("");
        require(success, "Payment transfer failed");
        
        emit NFTPurchased(_listingId, msg.sender, listing.seller, listing.price);
    }
    
    /**
     * @dev Cancel an active listing
     * @param _listingId ID of the listing to cancel
     */
    function cancelListing(uint256 _listingId) external nonReentrant {
        Listing storage listing = listings[_listingId];
        require(listing.active, "Listing not active");
        require(
            msg.sender == listing.seller || msg.sender == owner(),
            "Only seller or owner can cancel"
        );
        
        listing.active = false;
        nftToListing[listing.nftContract][listing.tokenId] = 0;
        
        emit ListingCancelled(_listingId);
    }
    
    /**
     * @dev Get listing details by listing ID
     * @param _listingId ID of the listing
     * @return Listing details
     */
    function getListing(uint256 _listingId) external view returns (Listing memory) {
        return listings[_listingId];
    }
    
    /**
     * @dev Get total number of listings created
     * @return Total listing count
     */
    function getTotalListings() external view returns (uint256) {
        return _listingIdCounter;
    }
    
    /**
     * @dev Withdraw marketplace earnings (only owner)
     */
    function withdrawEarnings() external onlyOwner {
        uint256 amount = marketplaceEarnings;
        require(amount > 0, "No earnings to withdraw");
        
        marketplaceEarnings = 0;
        
        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "Withdrawal failed");
    }
    
    /**
     * @dev Emergency function to withdraw stuck ETH (only owner)
     */
    function emergencyWithdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        (bool success, ) = payable(owner()).call{value: balance}("");
        require(success, "Emergency withdrawal failed");
    }
    
    // Function to receive ETH
    receive() external payable {}
   ![Contract Screenshot](./contract-screenshot.png)

- Contract Address: `0xeeC1C771B2F6d330215272b20621d948d0Ee0ed`
- Deployed using: Remix + MetaMask
- Network: Core Testnet2
