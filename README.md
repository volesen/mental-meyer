# Mental Meyer

[Meyer](https://da.wikipedia.org/wiki/Meyer_%28terningspil%29) is a Danish dice game about bluffing. You roll two dice under a cup, peek, and announce a value. The next player either believes you and rolls again, or lifts the cup to find out whether you lied.

We want to play it peer to peer, without trusting a dealer or another player's dice generator. Like [mental poker](https://people.csail.mit.edu/rivest/pubs/SRA81.pdf) and [coin flipping over a distance](https://en.wikipedia.org/wiki/Coin_flipping#Telecommunications), everyone helps generate the dice, only the roller can inspect them, and a challenge lets everyone check them. Bluffing stays legal.

## Plan of attack

A **commitment** locks in a secret value: publish its salted hash now, reveal the value and salt later. Others can verify the opening without being able to inspect the secret beforehand.

For each roll:

1. **Everyone commits.** Each active player chooses a uniform `share` in `0..35` and a random 32-byte secret `salt`. Publish a commitment. Fix the complete set before anyone reveals anything.
2. **Everyone except the roller opens.** Publish the shares and salts, and verify them against the commitments. The roller keeps their contribution secret.
3. **The roller declares.** Only the roller can calculate the dice. They announce a legal value, which may be a bluff.
4. **The next player decides.** Acceptance retires the roll unopened and starts a fresh roll for the next player. A challenge makes the roller reveal their contribution, so everyone can reconstruct the dice and apply the penalty.

The dice are just the combined contributions:

```text
r  = sum(all shares) mod 36
d1 = 1 + (r mod 6)
d2 = 1 + floor(r / 6)
```

Under the hash-commitment assumptions, one honest player's independent uniform contribution makes the underlying draw uniform over all 36 ordered outcomes. An honest roller's secret contribution hides the dice, except for information they disclose through play.

Cryptography cannot force someone to finish. Invalid required openings or refusal to reveal invoke an agreed forfeit policy, never a free reroll. Selective aborts can still bias which games finish. This sketch uses salted hash commitments in the random-oracle model and covers one private roll per turn.

## The CBOR wire format

Before play, agree on a fresh 32-byte game ID, player order, initial roller, lives, and exact rules: ranking, legal declarations, whether a claim means “exactly” or “at least,” turn progression, and penalties, including any special treatment of 32.

The session must authenticate senders and give everyone the same accepted message history. How peers establish those properties is outside this application format; CBOR supplies neither.

Each message is one array, using [core deterministic CBOR encoding](https://www.rfc-editor.org/rfc/rfc8949.html#section-4.2.1):

```text
[version, game, roll, player, action]
```

`version` is `1`. `game` identifies this game and its agreed configuration and must never be reused. `roll` starts at `0` and increments for every new draw. `player` is the authenticated sender's zero-based index in the initial roster; indices remain fixed after elimination.

The complete schema is in [protocol.cddl](protocol.cddl).

`rank` indexes the agreed ranking from weakest to strongest. With Lille Meyer:

```text
[32, 41, 42, 43, 51, 52, 53, 54, 61, 62, 63,
 64, 65, 11, 22, 33, 44, 55, 66, 31, 21]
```

Thus `[2, 20]` claims Meyer and `[2, 18]` claims a pair of sixes. To rank revealed dice, calculate `10 * max(d1, d2) + min(d1, d2)` and look up its index.

Let `E(value)` mean deterministic CBOR encoding. The exact commitment is:

```text
commitment = SHA256(E([
  "mental-meyer/commit", 1, game, roll, player, share, salt
]))
```

The domain string is CBOR text. `game`, `salt`, and the commitment are byte strings, not hex text. Hash the entire encoded array. An `OPEN` must reproduce the sender's commitment for that game and roll.

Use cryptographically secure randomness. Sample each share anew; repeated values are valid. For an unbiased sampler, draw a random byte until it is below `252`, then take it modulo `36`. Generate a fresh random salt for every contribution.

Clients enforce this order:

| Phase | Required messages |
| --- | --- |
| Commit | One `COMMIT` from every active player. |
| Open | One verified `OPEN` from everyone except the roller. |
| Declare | One legal `CLAIM` from the roller. |
| Decide | One final `ACCEPT` or `CHALLENGE` from the next player. |
| Challenge only | One verified `OPEN` from the roller. |

Complete each phase before advancing. Derive lives, the next roller, and challenged outcomes locally from the agreed rules; no `RESULT` message is needed.

For interoperable clients:

- Require exact array lengths, types, and ranges, shortest integer/length encodings, and definite lengths. Reject unknown versions/actions, maps, tags, floats, and trailing data within a message.
- Validate the game, roll, authenticated sender, active membership, phase, and permission for each action.
- Treat identical retransmissions as no-ops. Conflicting commitments, claims, or decisions are protocol violations.
- Keep the roller's opening secret until a challenge. Premature disclosure is a protocol violation; rejecting a message cannot undo the leak.
- Never recycle a roll ID or an earlier contribution. Invalid openings invoke the forfeit policy; missing messages invoke the session's abort policy.
