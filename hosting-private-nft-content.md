---
title: Hosting Private NFT Content with Bubble Protocol
description: NFTs typically employ off-chain storage to host their digital content. Decentralised networks like IPFS are commonly used, creating a fully decentralised solution. While this approach is fine for public NFT content it does not cater to private content that should only be accessible to NFT holders.
image: https://images.unsplash.com/photo-1580847097346-72d80f164702?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1920&q=80
image-caption: Photo by Tim Mossholder on Unsplash
---
NFTs typically employ off-chain storage to host their digital content. Decentralised networks like IPFS are commonly used, creating a fully decentralised solution. While this approach is fine for public NFT content it does not cater to private content that should only be accessible to NFT holders.

So, how can we make off-chain content available exclusively to NFT holders in a decentralised way? Enter Bubble Protocol.

## Bubble Protocol

Bubble is a general purpose protocol that enables smart contracts to control access to off-chain content. It is blockchain agnostic and supports EVM-based blockchains. Content is held in an off-chain bubble, a container of files and directories whose access permissions are governed by the bubble’s custom designed smart contract. The bubble is stored on a relay service of the user’s or developer’s choosing, which could be a cloud service, a decentralised storage network or even a private server.

For a smart contract to manage an off-chain bubble, it needs to provide just a single method:

```javascript
function getAccessPermissions( address user, uint256 fileId ) external view returns (uint256);
```

This method is invoked by the relay service whenever anyone tries to access the bubble controlled by your smart contract. Given a user’s public address and the ID of the file within the bubble it returns the user’s read, write, append and even execute permissions for that file.

Let’s see how this applies to NFT private content with an example.

## An NFT Example

We’ll define some high-level requirements for our off-chain content. We want:

1. A directory for each NFT token that’s accessible only by the holder of that token.
2. A public directory that anyone can access to read public metadata, images, etc.
3. The NFT creator must NOT be able to overwrite or delete any content after the NFT has been issued.

Here’s a smart contract that encapsulates those requirements:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "AccessControlledStorage.sol";
import "AccessControlBits.sol";
import "BubbleStandardContent.sol";
import "@openzeppelin/contracts/interfaces/IERC721.sol";

contract ERC721ControlledBubble is AccessControlledStorage {

    IERC721 public nftContract;
    address owner = msg.sender;
    bool public locked = false;

    constructor(IERC721 nft) {
      nftContract = nft;
    }

   function getAccessPermissions( address user, uint256 contentId ) override external view returns (uint256) {
      if (user == owner) {
          if (!locked) return DIRECTORY_BIT | READ_BIT | WRITE_BIT | APPEND_BIT;
          else return DIRECTORY_BIT | READ_BIT;
      }
      else if (contentId == 0) return NO_PERMISSIONS;
      else if (contentId == PUBLIC_DISCOVERY_DIRECTORY) return DIRECTORY_BIT | READ_BIT;
      else if (nftContract.ownerOf(contentId) == user) return DIRECTORY_BIT | READ_BIT;
      else return NO_PERMISSIONS;
    }

    function lock() external {
        require(msg.sender == owner, 'permission denied');
        locked = true;
    }
    
}
```

The smart contract takes an ERC721 contract to control access to the bubble. Each token holder is allocated a read-only directory named after the token id, and there’s a single public directory that anyone can read. The creator has total control over the bubble content until they transition the contract with the lock function, after which they’re limited to read-only access.

Clearly, this smart contract could be expanded to add other features, such as an append-only directory to let token holders send messages to the creator, or could be integrated directly into the NFT contract itself.

Once deployed to a blockchain, an off-chain bubble can be created. This can be accomplished in code using the [Bubble SDK](https://github.com/Bubble-Protocol/bubble-sdk), but we’ll use the [Bubble Tools](https://github.com/Bubble-Protocol/bubble-tools) command-line interface:

```
bubble content create-bubble https://vault.bubbleprotocol.com/v2/ethereum 0x405ff917b5973c2e9db88546f4dd5fd82f87de32
```

Now, content can be written to the bubble:

```
bubble content write https://vault.bubbleprotocol.com/v2/ethereum 0x405ff917b5973c2e9db88546f4dd5fd82f87de32 1/image token1.jpg

> eyJjaGFpbiI6MSwiY29udHJhY3QiOiIweDQwNWZmOTE3YjU5NzNjMmU5ZGI4ODU0NmY0ZGQ1ZmQ4MmY4N2RlMzIiLCJwcm92aWRlciI6Imh0dHBzOi8vdmF1bHQuYnViYmxlcHJvdG9jb2wuY29tL3YyL2V0aGVyZXVtIiwiZmlsZSI6IjB4MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMS9pbWFnZSJ9
```

Once the contract is locked, decentralised applications can read the content using the ContentManager in the Bubble SDK. The user’s digital signature in the read request authenticates the user, permitting access:

```javascript
const accounts = await window.ethereum.getAccounts();

const signFunction = (hash) => {
  return window.ethereum.request({
    method: 'personal_sign',
    params: [hash, accounts[0], 'Bubble content request'],
  })
  .then(toEthereumSignature);
}

const tokenImageContentId = "eyJjaGFpbiI6MSwiY29udHJhY3QiOiIweDQwNWZmOTE3YjU5NzNjMmU5ZGI4ODU0NmY0ZGQ1ZmQ4MmY4N2RlMzIiLCJwcm92aWRlciI6Imh0dHBzOi8vdmF1bHQuYnViYmxlcHJvdG9jb2wuY29tL3YyL2V0aGVyZXVtIiwiZmlsZSI6IjB4MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMS9pbWFnZSJ9";

const image = await ContentManager.read(tokenImageContentId, signFunction);
```

## Trust in Off-Chain Storage

Alongside any encryption strategy employed by users, the integrity, security, and availability of off-chain data relies heavily on the reliability of the service hosting it. As such, users are required to have a certain level of trust in their chosen off-chain storage service.

However, the design of Bubble Protocol inherently minimises the extent of this trust. By decentralising access control logic and encouraging competition among storage providers, Bubble Protocol allows users the freedom to choose the storage service that they find most reliable and trustworthy. This also gives users the flexibility to switch to a different service should their confidence in the current service be compromised, or to use multiple services.

In the context of our NFT example, the optimal choice for hosting NFT data would be a decentralised storage network. Here at Bubble Protocol, we’re in the process of developing such a network, one that has our access control protocol seamlessly integrated.

<p align="center">. . .</p>

As we’ve shown, Bubble Protocol offers a simple and versatile solution for hosting private NFT content, and for web3 in general. It provides a secure and decentralised way to manage access to off-chain content, whether it’s for an individual token holder or accessible to the wider public.

To learn more about the Bubble Protocol, please visit our website, check out our GitHub, or join our community on Discord. You can also follow us on Twitter for the latest updates.

[Website](https://bubbleprotocol.com) | [Github](https://github.com/bubble-protocol) | [Discord](https://discord.gg/sSnvK5C) | [Twitter](https://twitter.com/bubbleprotocol)