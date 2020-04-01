# ADR 56: Proving Amnesia Attack

## Changelog

- 02.04.20: Initial Draft

## Context

Whilst most created evidence of malicious behaviour is self evident such that any individual can verify them independently there are types of evidence that require collaboration from the network of validators, known as global evidence, in order to accumulate enough information to create evidence that is individually verifiable and can therefore be processed through consensus. Fork Accountability as a whole constitutes of detection, proving and punishing. This ADR addresses how to prove infringement from global evidence.

So far, only [flip flopping](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md#flip-flopping) attacks such as amnesia (which will be what the reactor initially focuses on) and back to the past (only causes light forks) have been identified as forms of global evidence that require more information from other validators.

## Decision

A new reactor will perform the task of turning global evidence to individual evidence. This special circumstance reactor could be combined with the current evidence reactor although the evidence reactor is currently used for gossiping and storing of evidence and so it's purpose would expand. The reactor is initialized when a full node receives PotentialAmnesiaEvidence from a light client.

*REMARK: This means that only the set of full nodes that are directly connected to the light client that observed the misbehaviour will process the evidence. Is this sufficient, do we want to gossip to further nodes, do we want just a single node to process evidence to avoid too much communication overhead?*

```
type PotentialAmnesiaEvidence struct {
	C1 types.Commit
	C2 types.Commit
}
```

The receiving full node first runs `Verify()` to ensure that the evidence is valid i.e. both `Commit`s are valid and are of the same height.

*NOTE: This means that the commits should be within the bonding period minus some margin as well as within the expected range of heights wherein votesets are kept*

The suspect group are the overlapping set of validators that have voted for both `C1` and `C2`.

The full node requests `VoteSet`'s from all validators in that validator set. A configurable time or height is set with which validators must respond. If a validator, that is part of the suspect group, fails to provide a vote, we mark them as faulty.

*REMARK: How do we form authentic evidence of accused nodes failing to respond in the given time?*

From these received vote sets, the node constructs a vote set that each of the suspect group *sent* (as well as having what votes they received). With both the votes sent and received if any of the following is found, the validator is marked faulty:

1. More than one PREVOTE message sent in a round
2. More than one PRECOMMIT message sent in a round
3. PRECOMMIT message is sent without receiving 2/3+ PREVOTE messages
4. PREVOTE message is sent for the value V’ in round R’ and the PRECOMMIT message had been sent for the value V in round R by the same process (R’ > R) and there are no 2/3+ PREVOTE(VR, V’) messages (VR ≥ 0 and VR ≥ R and VR < R’) as the justification for sending PREVOTE(R’, V’)

*REMARK: With this last item, does this imply that all vote sets for rounds between and including `C1` and `C2` must
 be sent?*

From this, `PotentialAmnesiaEvidence` is broken down into individually verifiable evidence for each of the validators and sent to evidence reactor for gossiping:

- 1 & 2 -> `DuplicateVoteEvidence`
- 3 & 4 -> `AmnesiaEvidence`

```
type AmnesiaEvidence struct {
	VoteSets []*types.VoteSet
	Vote	 *types.Vote
}
```

Until the reactor processes the evidence itself, or evidence of the same infringement is proposed by another node, the node does not participate in consensus.

*REMARK: We probably don't want all nodes to be running this reactor as this would lead to unnecessary messages but we might want to gossip to other nodes to make them aware of the fork so perhaps we want to gossip the evidence whilst telling them not to bother running it until some time has expired.*

As a slight alternative, `PotentialAmnesiaEvidence` could be first gossiped, proposed and then committed through the consensus algorithm, and then the reactor run **after** the evidence is committed to the chain by the validator set involved in consensus for that height. Then following this, `AmnesiaEvidence` can be proposed and committed.

## Status

Proposed

## Consequences

### Positive

Increasing fork detection makes the system more secure

### Negative

Potentially a large network load to prove the evidence.

Non-responsive nodes that are part of the suspect group will be punished

### Neutral

## References

- [Fork accountability algorithm](https://docs.google.com/document/d/11ZhMsCj3y7zIZz4udO9l25xqb0kl7gmWqNpGVRzOeyY/edit)
- [Fork accountability spec](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md)
