# nyampahbersama
**Trust Framework for Social Blockchain and Collective Memory**  ---  EU Login: n00jcciv  Author: Muhamad Oky Muhlisin  Afiliasi : Komunitas Nyampah Bersama


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

/**
 * @title 2BCycleSoulboundArchive
 * @dev Modular NFT + Vault + Soul + Story
 *   - Setiap token adalah bab hidup, punya jiwa, narasi, dan histori verifiable
 *   - Vault token dapat dibuka hanya jika rantai NFT/proof terpenuhi
 *   - Kontrak ini bergerak sebagai "museum kehidupan" & generator nilai digital baru
 */
contract CycleSoulArchive is ERC721URIStorage, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIds;

    // ====== 🫀 JIWA KONTRAK: MANIFESTO, NARASI, HARAPAN ======
    string public SOUL_MANIFESTO;
    string public MISSION_STATEMENT;

    // ====== 🗝️ VAULT CONFIGURATION (KUNCI TOKEN BERHARGA) ======
    address public vaultTokenAddress;
    uint256 public vaultTokenAmount;
    bool public vaultLocked;
    mapping(uint256 => bool) public isTokenUsedForVault;

    // ====== 🧬 LINKED NFT MEMORY TREE ======
    mapping(uint256 => uint256[]) public linkedNFTs; // modular link graph (tokenId => [tokenId,...])

    // ====== 📝 AKSI, PROOF, CATATAN HISTORI ======
    struct ActionProof {
        string actionType;
        string description;
        address actor;
        uint256 timestamp;
    }
    mapping(uint256 => ActionProof[]) public tokenProofs; // every NFT: chains of action

    event ManifestoWritten(address creator, string manifesto, string mission);
    event SoulNFTMinted(uint256 tokenId, address to, string metadataURI, string story);
    event ActionLogged(uint256 tokenId, address actor, string actionType, string desc, uint256 time);
    event VaultUnlocked(address by, uint256 atBlock, uint256[] fullProofNFTs);

    constructor(
        string memory initialManifesto,
        string memory mission
    ) ERC721("2BCycle Modular Soul NFT", "2BCSoul") {
        SOUL_MANIFESTO = initialManifesto;
        MISSION_STATEMENT = mission;
        vaultLocked = true;

        emit ManifestoWritten(msg.sender, initialManifesto, mission);
    }

    // ====== FUNGSI JIWA DAN NARASI ======

    function updateManifesto(string memory newManifesto) external onlyOwner {
        require(bytes(newManifesto).length > 0, "Manifesto cannot be blank");
        SOUL_MANIFESTO = newManifesto;
        emit ManifestoWritten(msg.sender, newManifesto, MISSION_STATEMENT);
    }

    function updateMission(string memory newMission) external onlyOwner {
        require(bytes(newMission).length > 0, "Mission cannot be blank");
        MISSION_STATEMENT = newMission;
        emit ManifestoWritten(msg.sender, SOUL_MANIFESTO, newMission);
    }

    // ====== MODULAR NFT (ERC721 + LOGIKA) ======

    function mintSoulNFT(
        address to,
        string memory metadataURI,
        string memory story,
        uint256[] memory _linkedNFTs
    ) external onlyOwner returns (uint256) {
        _tokenIds.increment();
        uint256 newTokenId = _tokenIds.current();

        _mint(to, newTokenId);
        _setTokenURI(newTokenId, metadataURI);

        // Linked NFT
        if (_linkedNFTs.length > 0) {
            for (uint i = 0; i < _linkedNFTs.length; i++) {
                linkedNFTs[newTokenId].push(_linkedNFTs[i]);
            }
        }

        // Rekam catatan narasi mint
        tokenProofs[newTokenId].push(ActionProof({
            actionType: "mint",
            description: story,
            actor: to,
            timestamp: block.timestamp
        }));

        emit SoulNFTMinted(newTokenId, to, metadataURI, story);
        return newTokenId;
    }

    // ====== AKSI & PROOF ======

    function logAction(
        uint256 tokenId,
        string memory actionType,
        string memory desc
    ) public {
        require(_exists(tokenId), "Not exist");
        tokenProofs[tokenId].push(ActionProof({
            actionType: actionType,
            description: desc,
            actor: msg.sender,
            timestamp: block.timestamp
        }));
        emit ActionLogged(tokenId, msg.sender, actionType, desc, block.timestamp);
    }

    // ====== VAULT: KUNCI, BUKA, DAN AMANKAN TOKEN ======

    function configureVault(address _token, uint256 _amount) external onlyOwner {
        vaultTokenAddress = _token;
        vaultTokenAmount = _amount;
        vaultLocked = true;
    }

    function unlockVault(uint256[] calldata owningNFTIds) external {
        require(vaultLocked, "Vault already unlocked");
        // Syarat: User harus pegang seluruh NFT yang dijadikan proof!
        for (uint i = 0; i < owningNFTIds.length; i++) {
            require(ownerOf(owningNFTIds[i]) == msg.sender, "You must own all proof NFTs");
            isTokenUsedForVault[owningNFTIds[i]] = true;
        }
        vaultLocked = false;
        emit VaultUnlocked(msg.sender, block.number, owningNFTIds);
        // Eksekusi transfer token vault ke pemenang (manual/onsend)
    }

    // ====== Lihat histori/proof NFT ======

    function getActionProofHistory(uint256 tokenId) public view returns (ActionProof[] memory) {
        return tokenProofs[tokenId];
    }

    // ====== Ekstensi: Modular Logic, Linked NFT, Chain of Memory ======
    function getLinkedNFTs(uint256 tokenId) public view returns (uint256[] memory) {
        return linkedNFTs[tokenId];
    }
}
