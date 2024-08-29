## `public` functions not used internally could be marked `external`

## Vulnerability Details

Functions not used within a contract marked `public` should be marked `external` as this will help with gas optimization.

27 Found Instances
Found in src/Cred.sol Line: 95

    function version() public pure returns (uint256) {
Found in src/Cred.sol Line: 182

    function sellShareCred(uint256 credId_, uint256 amount_, uint256 minPrice_) public {
Found in src/Cred.sol Line: 186

    function buyShareCredFor(uint256 credId_, uint256 amount_, address curator_, uint256 maxPrice_) public payable {
Found in src/Cred.sol Line: 191

    function batchBuyShareCred(
Found in src/Cred.sol Line: 213

    function batchSellShareCred(
Found in src/Cred.sol Line: 232

    function createCred(
Found in src/Cred.sol Line: 399

    function getCreatorRoyalty(uint256 credId_) public view returns (uint16 buyShareRoyalty, uint16 sellShareRoyalty) {
Found in src/Cred.sol Line: 440

    function getCuratorAddresses(
Found in src/Cred.sol Line: 464

    function getCuratorAddressesWithAmount(
Found in src/Cred.sol Line: 531

    function getRoot(uint256 credId_) public view returns (bytes32) {
Found in src/PhiFactory.sol Line: 60

    function version() public pure returns (uint256) {
Found in src/PhiFactory.sol Line: 64

    function contractURI(address nftAddress) public view returns (string memory) {
Found in src/PhiFactory.sol Line: 541

    function checkProof(bytes32[] calldata proof, bytes32 leaf, bytes32 root) public pure returns (bool) {
Found in src/art/PhiNFT1155.sol Line: 68

    function version() public pure returns (uint256) {
Found in src/art/PhiNFT1155.sol Line: 216

    function supportsInterface(bytes4 interfaceId)
Found in src/art/PhiNFT1155.sol Line: 227

    function contractURI() public view returns (string memory) {
Found in src/art/PhiNFT1155.sol Line: 234

    function uri(uint256 tokenId_) public view override returns (string memory) {
Found in src/art/PhiNFT1155.sol Line: 241

    function uri(uint256 tokenId_, address minter_) public view returns (string memory) {
Found in src/art/PhiNFT1155.sol Line: 251

    function getPhiFactoryContract() public view override returns (IPhiFactory) {
Found in src/art/PhiNFT1155.sol Line: 255

    function getTokenIdFromFactoryArtId(uint256 artId_) public view returns (uint256 tokenId) {
Found in src/art/PhiNFT1155.sol Line: 259

    function getFactoryArtId(uint256 tokenId_) public view override(Claimable, IPhiNFT1155) returns (uint256) {
Found in src/art/PhiNFT1155.sol Line: 263

    function getArtDataFromFactory(uint256 artId_) public view returns (IPhiFactory.ArtData memory) {
Found in src/art/PhiNFT1155.sol Line: 307

    function safeTransferFrom(
Found in src/art/PhiNFT1155.sol Line: 332

    function safeBatchTransferFrom(
Found in src/curve/BondingCurve.sol Line: 51

    function getPriceData(
Found in src/curve/BondingCurve.sol Line: 86

    function getBuyPriceAfterFee(uint256 credId_, uint256 supply_, uint256 amount_) public view returns (uint256) {
Found in src/curve/BondingCurve.sol Line: 106

    function getSellPriceAfterFee(uint256 credId_, uint256 supply_, uint256 amount_) public view returns (uint256) {