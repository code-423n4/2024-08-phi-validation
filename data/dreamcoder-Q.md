In CreatorRoyaltiesControl abstract

bool private initilaized;
function getRoyalties(uint256 tokenId) public view returns (RoyaltyConfiguration memory) {
        if (!initilaized) revert NotInitialized();
        // Function body
    }

1. The variable "initilaized" should be corrected to "initialized". 
2. It's more professional and improve readability to use modifier

Improved code
bool private initialized;
modifier onlyInitialized() {
    if (!initialized) revert NotInitialized();
    _;
}
function getRoyalties(uint256 tokenId) public view onlyInitialized returns (RoyaltyConfiguration memory) {
    // Function body
}
