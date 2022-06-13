# ERC-721 DAO

In this repository you can find an ERC-721 DAO. There's an array with the ID of the valid tokens / NFTs that a user can use. NFT holders can create and vote on a proposal.

The administrator can also count the votes to settle the decision of the proposal, add or delete a token ID for the valid NFTs array and change the minimum quorum for the proposals to be accepted.

This project was based on a DAO built by Moralis. I've worked on the gas optimization, on a function to delete tokens from the valid tokens array and on the creation of a new concept: the quorum system.

## Base of our code 👨‍💻

The following code represents the base of all the project.

We're creating a constructor which is used to define the owner, the array of valid tokens and the quorum value. In this case, I used one of my NFTs minted on the Mumbai testnet as an example, but I could also not push any element to the array during the constructor if I opted for not hard coding it.

A simple onlyOwner modifier is being created to restrict the use of some functions only to the administrator.

A struct was also created for managing the proposal datails.

There are also three different events which may be used by a front-end.

Gas optimization was taken into account while developing the base of our code.

```
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

interface IDaoContract {
    function balanceOf(address, uint) external view returns (uint);
}

contract Dao {

    address public owner;
    uint8 public quorum;
    uint nextProposal;
    uint[] public validTokens;
    IDaoContract daoContract;

    constructor(uint8 _quorum) {
        require(_quorum <= 100, "Minimum quorum should not be above 100%");
        owner = msg.sender;
        quorum = _quorum;
        nextProposal = 1;
        daoContract = IDaoContract(0x2953399124F0cBB46d2CbACD8A89cF0599974963);
        validTokens = [90589094416039184604703704186156568307524264362799260266890565735654715555850]; //ID of my NFT
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "You're not the owner");
        _;
    }

    struct proposal {
        uint256 id;
        uint256 deadline;
        uint256 votesUp;
        uint256 votesDown;
        uint256 maxVotes;
        address[] canVote;
        string description;        
        mapping(address => bool) voteStatus;
        bool exists;
        bool countCounducted;
        bool passed;
    }

    mapping(uint => proposal) public Proposals;

    event proposalCreated (
        address proposer,
        string description,
        uint256 id,
        uint256 maxVotes
    );

    event newVote (
        uint256 votesUp,
        uint256 votesDown,
        uint256 proposal,
        address voter,
        bool votedFor
    );

    event proposalCount (
        uint256 id,
        bool passed
    );
```

## Functions for the holders 🏃

The functions of the code that are designed for the holders are:
* createProposal: you can create a proposal if you hold one of the NFTs from the array.
* voteOnProposal: you can vote on the proposal you want

There are also two private functions: checkProposalEligibility and checkVoteEligibility, which assert that a user can create a proposal and vote.

```
function checkProposalEligibility(address _proposalAddr) private view returns(bool) {
        for (uint i = 0; i < validTokens.length; i++) {
            if (daoContract.balanceOf(_proposalAddr, validTokens[i]) >= 1) {
                return true;
            }
        }
        return false;
    }

    function checkVoteEligibility(uint _id, address _voter) private view returns(bool) {
        for (uint i = 0; i < Proposals[_id].canVote.length; i++) {
            if (Proposals[_id].canVote[i] == _voter) {
                return true;
            }
        }
        return false;
    }

    function createProposal(string memory _description, address[] memory _canVote) public {
        require(checkProposalEligibility(msg.sender), "You must own a NFT to create a proposal");

        proposal storage newProposal = Proposals[nextProposal];
        newProposal.id = nextProposal;
        newProposal.exists = true;
        newProposal.description = _description;
        newProposal.deadline = block.number + 3000; // More or less 12 hours
        newProposal.canVote = _canVote;
        newProposal.maxVotes = _canVote.length;

        emit proposalCreated(msg.sender, _description, nextProposal, _canVote.length);
        nextProposal++;
    }

    function voteOnProposal(uint _id, bool _vote) public {
        require(Proposals[_id].exists, "This proposal doesn't exist");
        require(checkVoteEligibility(_id, msg.sender), "You can't vote on this proposal");
        require(!Proposals[_id].voteStatus[msg.sender], "You've already voted");
        require(block.number <= Proposals[_id].deadline, "The deadline has passed");

        proposal storage p = Proposals[_id];

        if(_vote) {
            p.votesUp++;
        } else {
            p.votesDown++;
        }

        p.voteStatus[msg.sender] = true;

        emit newVote(p.votesUp, p.votesDown, _id, msg.sender, _vote);
    }
```

## Functions for the administrator 👑

There are four functions available for the administrator:
* countVotes: settles the result of a proposal
* addTokenId: adds a new token to the validTokens array
* deleteToken: deletes a token from the valiedTokens array
* changeQuorum: changes the minimum quorum of all the proposals

A proposal is approved/declined when the administrator counts the votes. There must be more upvotes than downvotes and the quorum must be achieved for the approval.

```
    function countVotes(uint _id) public onlyOwner {
        require(Proposals[_id].exists, "This proposal doesn't exist");
        require(block.number > Proposals[_id].deadline, "You must wait for the deadline");
        require(!Proposals[_id].countCounducted, "You've already counted the votes");

        proposal storage p = Proposals[_id];

        if (Proposals[_id].votesDown < Proposals[_id].votesUp &&
           (Proposals[_id].votesUp + Proposals[_id].votesDown) / Proposals[_id].maxVotes >= quorum / 100) {
            Proposals[_id].passed = true; 
        }

        p.countCounducted = true;

        emit proposalCount(_id, p.passed);
    }

    function addTokenId(uint _tokenId) public onlyOwner {
        validTokens.push(_tokenId);
    }

    function deleteToken(uint _index) public onlyOwner {
        validTokens[_index] = validTokens[validTokens.length - 1];
        validTokens.pop();
    }

    function changeQuorum(uint8 _quorum) public onlyOwner {
        require(_quorum <= 100, "Minimum quorum should not be above 100%");
        quorum = _quorum;
    }
```

## DAO Deployment 📜
A deploy script was written in JavaScript. It deploys our code using Hardhat.

```
const hre = require("hardhat");

async function main() {

  const Dao = await hre.ethers.getContractFactory("Dao");
  const dao = await Dao.deploy();

  await dao.deployed();

  console.log("Dao deployed to:", dao.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## Thanks 😄

Thanks for reading my code! This is the first time I've tried to code a DAO! I can easily notice my evolution at each project.

If you would like to talk to me, feel free to text me on Twitter or LinkedIn.
